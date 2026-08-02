# 04-CoreDNS Troubleshooting: Service Domain Resolution, Upstream DNS, and nslookup within Pods

Recommended path:

    04-Kubernetes/08-Operations/03-Cluster Basic Troubleshooting/04-CoreDNS Troubleshooting: Service Domain Resolution, Upstream DNS, and nslookup within Pods.md

Tags:

    #Kubernetes
    #CoreDNS
    #DNS
    #ServiceFoundOut.
    #ServiceParsing
    #PodNetwork
    #ClusterInfrastructureBarriers

---

## I. Document Description

This document records basic troubleshooting methods for CoreDNS-related issues in Kubernetes clusters.

CoreDNS is the internal DNS component of Kubernetes clusters, primarily responsible for:

    1. Resolving Service domain names within Pods
    2. Resolving Headless Service domain names within Pods
    3. Resolving external domain names within Pods
    4. Kubernetes service discovery
    5. Forwarding cluster external domain names to upstream DNS

Common issues:

    1. Unable to resolve Service domain names within Pods
    2. Unable to resolve external domain names within Pods
    3. nslookup kubernetes.default fails
    4. Service is normal but domain access fails
    5. CoreDNS Pod in CrashLoopBackOff state
    6. CoreDNS logs show timeout
    7. Upstream DNS configuration error
    8. StatefulSet Headless Service domain resolution anomaly
    9. Ingress/Gateway backend service name resolution anomaly

Document objectives:

    1. Understand the relationship between CoreDNS, Service, and Pod DNS
    2. Master DNS verification methods within Pods
    3. Know how to check CoreDNS Pod status
    4. Know how to check kube-dns Service
    5. Know how to check CoreDNS ConfigMap
    6. Know how to troubleshoot Service domain unavailability
    7. Know how to troubleshoot external domain resolution failure
    8. Know how to troubleshoot Headless Service resolution issues
    9. Establish a standard DNS troubleshooting path

Applicable scenarios:

    1. kubeadm self-built cluster
    2. Private Kubernetes cluster
    3. Pod fails to access Service domain
    4. Pod fails to access external domain
    5. CoreDNS anomaly
    6. Service discovery anomaly
    7. StatefulSet domain resolution anomaly

---

## II. CoreDNS Role in Kubernetes

Common internal domain formats in Kubernetes:

    <service-name>.<namespace>.svc.cluster.local

Example:

    kubernetes.default.svc.cluster.local
    nginx-demo.default.svc.cluster.local
    mysql.mysql.svc.cluster.local

When accessing Service within Pods, the following can typically be used:

    nginx-demo
    nginx-demo.default
    nginx-demo.default.svc
    nginx-demo.default.svc.cluster.local

CoreDNS is responsible for resolving these domain names into Service ClusterIP or backend Pod IP.

---

## III. DNS Access Chain

When a Pod accesses a Service domain, the access chain is roughly as follows:

    Pod
     |
     v
    /etc/resolv.conf
     |
     v
    kube-dns Service ClusterIP
     |
     v
    CoreDNS Pod
     |
     v
    Kubernetes API / Service records
     |
     v
    Return Service ClusterIP

When a Pod accesses an external domain:

    Pod
     |
     v
    /etc/resolv.conf
     |
     v
    kube-dns Service ClusterIP
     |
     v
    CoreDNS Pod
     |
     v
    CoreDNS forward plugin
     |
     v
    Upstream DNS
     |
     v
    Return external domain resolution result

---

## IV. Relationship Between CoreDNS and kube-dns Service

In Kubernetes, the service name is typically still called:

    kube-dns

But the backend Pod is usually:

    CoreDNS

Check kube-dns Service:

    kubectl -n kube-system get svc kube-dns

Example:

    NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)
    kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP

Explanation:

    kube-dns is the Service name.
    CoreDNS is the Pod actually handling DNS requests.
    The Pod's /etc/resolv.conf typically points to the ClusterIP of the kube-dns Service.

---

## V. Troubleshooting Overview

DNS troubleshooting is recommended to follow this order:

    1. First confirm if CoreDNS Pod is Running
    2. Check if kube-dns Service exists
    3. Check if kube-dns Endpoints are empty
    4. Check /etc/resolv.conf within the Pod
    5. Perform nslookup kubernetes.default within the Pod
    6. Perform nslookup on business Service within the Pod
    7. Compare if direct access to Service ClusterIP is normal
    8. Check CoreDNS ConfigMap
    9. Check CoreDNS logs
    10. Troubleshoot upstream DNS
    11. Troubleshoot Service / Endpoints / NetworkPolicy / CNI

Troubleshooting branches: /think

# DNS Not Working

    |
    |-- Service Domain Not Reachable
    |       |
    |       |-- CoreDNS Abnormality
    |       |-- kube-dns Service Abnormality
    |       |-- Service Does Not Exist
    |       |-- Namespace Error
    |       |-- Endpoints Are Empty
    |
    |-- External Domain Not Reachable
            |
            |-- CoreDNS Forward Upstream Error
            |-- Node /etc/resolv.conf Error
            |-- Upstream DNS Not Reachable
            |-- Network Policy or Firewall Block
            |-- CoreDNS Timeout to Upstream DNS

---

## Six. Step 1: Check CoreDNS Pod Status

Execute:

    kubectl -n kube-system get pods -o wide | grep coredns

Or:

    kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide

Normal status:

    Running
    READY 1/1

Example:

    coredns-xxxx   1/1   Running   0   10d

If status is abnormal:

    Pending
    CrashLoopBackOff
    ImagePullBackOff
    Error
    ContainerCreating

Need to troubleshoot CoreDNS Pod itself first.

---

## Seven. Step 2: Check kube-dns Service

Execute:

    kubectl -n kube-system get svc kube-dns

Check detailed information:

    kubectl -n kube-system describe svc kube-dns

Focus on:

    1. ClusterIP
    2. Port 53/UDP
    3. Port 53/TCP
    4. Selector
    5. Endpoints

Example:

    IP: 10.96.0.10
    Port: dns 53/UDP
    Port: dns-tcp 53/TCP
    Endpoints: 10.244.0.10:53,10.244.1.12:53

If Endpoints are empty, it means kube-dns Service has no backend CoreDNS Pod.

---

## Eight. Step 3: Check kube-dns Endpoints

Check:

    kubectl -n kube-system get endpoints kube-dns

Or:

    kubectl -n kube-system get ep kube-dns

Normal example:

    NAME       ENDPOINTS
    kube-dns   10.244.0.10:53,10.244.1.12:53

Abnormal example:

    NAME       ENDPOINTS
    kube-dns   <none>

If Endpoints are empty, prioritize checking:

    1. CoreDNS Pod Status
    2. CoreDNS Pod Label Matches Service Selector
    3. CoreDNS Pod Readiness
    4. kube-dns Service Selector Abnormality

Check Service selector:

    kubectl -n kube-system get svc kube-dns -o yaml | grep -A10 selector

Check CoreDNS Pod label:

    kubectl -n kube-system get pods --show-labels | grep coredns

---

## Nine. Step 4: Create DNS Test Pod

Recommend preparing a dedicated temporary Pod for DNS troubleshooting.

Create:

    kubectl run dns-test \
      --image=busybox:1.36 \
      --restart=Never \
      -- sleep 3600

Check:

    kubectl get pod dns-test -o wide

Enter Pod:

    kubectl exec -it dns-test -- sh

If temporary Pod fails to start, need to troubleshoot:

    1. Image Pull
    2. Node Status
    3. CNI
    4. Pod Pending

---

## Ten. Step 5: Check resolv.conf in Pod

In dns-test Pod execute:

    cat /etc/resolv.conf

Normal example:

    nameserver 10.96.0.10
    search default.svc.cluster.local svc.cluster.local cluster.local
    options ndots:5

Focus on:

    nameserver
        Should be kube-dns Service ClusterIP.

    search
        Should include current namespace and cluster.local search domains.

    ndots
        Kubernetes default is typically 5.

If nameserver is not kube-dns ClusterIP, need to check:

    1. kubelet DNS Configuration
    2. Cluster DNS Service IP
    3. Pod dnsPolicy
    4. Pod dnsConfig

---

## Eleven. Step 6: Test Kubernetes Default Service Resolution

In dns-test Pod execute:

    nslookup kubernetes.default

Or:

    nslookup kubernetes.default.svc.cluster.local

Should resolve to Kubernetes Service ClusterIP.

Check Kubernetes Service:

    kubectl get svc kubernetes -n default

Example: /think

NAME         TYPE        CLUSTER-IP   PORT(S)
kubernetes   ClusterIP   10.96.0.1    443/TCP

If `nslookup kubernetes.default` fails, it indicates abnormal internal DNS capabilities in the cluster.

---

## Twelve. Seventh Step: Test Business Service Resolution

Assume a business Service:

    namespace: default
    service: nginx-demo

Check the Service:

    kubectl get svc nginx-demo -n default

Test short domain:

    nslookup nginx-demo

Test with namespace:

    nslookup nginx-demo.default

Test full domain:

    nslookup nginx-demo.default.svc.cluster.local

If the full domain can resolve but the short domain cannot, check the namespace and search domain where the Pod resides.

---

## Thirteen. Service Domain Resolution Failure Troubleshooting

### 13.1 Check if the Service Exists

Execute:

    kubectl get svc -n default

Check the specific Service:

    kubectl get svc nginx-demo -n default

If the Service does not exist, DNS will certainly fail to resolve the Service domain.

---

### 13.2 Check if the Namespace is Correct

Service domain includes the namespace.

For example, if the Service is in `app-prod`:

    nginx-demo.app-prod.svc.cluster.local

If accessed from a Pod in the `default` namespace:

    nginx-demo

Will default resolve to:

    nginx-demo.default.svc.cluster.local

This will fail.

Correct access methods:

    nginx-demo.app-prod

Or:

    nginx-demo.app-prod.svc.cluster.local

---

### 13.3 Check Service Type

Ordinary ClusterIP Service:

    nslookup <service>.<namespace>.svc.cluster.local

Typically resolves to ClusterIP.

Headless Service:

    clusterIP: None

Typically resolves to backend Pod IP list.

Check:

    kubectl get svc <service-name> -n <namespace> -o yaml | grep clusterIP

---

### 13.4 Check Endpoints

DNS resolving the Service does not guarantee business connectivity.

If the Service is an ordinary ClusterIP, DNS typically returns ClusterIP.

But if Endpoints are empty, access still fails.

Check:

    kubectl get endpoints <service-name> -n <namespace>

    kubectl get endpointslice -n <namespace> -l kubernetes.io/service-name=<service-name>

If Endpoints are empty, continue troubleshooting:

    selector
    Pod labels
    ReadinessProbe
    Pod Ready status

Reference:

    03-Service Normal but Business Unreachable: selector, Endpoints, Ports, and kube-proxy Troubleshooting.md

---

## Fourteen. Test External Domain Resolution

Execute inside the `dns-test` Pod:

    nslookup www.baidu.com

Or:

    nslookup www.example.com

If internal Service can resolve but external domains cannot, the issue is likely:

    1. CoreDNS forward upstream DNS
    2. Node /etc/resolv.conf
    3. Upstream DNS unreachable
    4. Firewall blocking DNS
    5. Network policy blocking CoreDNS outbound

---

## Fifteen. View CoreDNS ConfigMap

CoreDNS configuration is stored in a ConfigMap.

Execute:

    kubectl -n kube-system get cm coredns

View the configuration:

    kubectl -n kube-system get cm coredns -o yaml

Or only view Corefile:

    kubectl -n kube-system get cm coredns -o jsonpath='{.data.Corefile}'

Common configuration example:

    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }

Focus on:

    kubernetes cluster.local
    forward . /etc/resolv.conf
    cache
    loop
    reload
    loadbalance

---

## Sixteen. Troubleshoot CoreDNS forward Upstream DNS

Common CoreDNS configuration:

    forward . /etc/resolv.conf

Meaning:

    CoreDNS forwards non-cluster.local external domain requests to the DNS configured in the /etc/resolv.conf of the node where the CoreDNS Pod resides.

If the node's /etc/resolv.conf configuration is abnormal, internal and external domain resolution in Pods may fail.

---

### 16.1 Check the Node Where CoreDNS Resides

Execute: /think

kubectl -n kube-system get pod -l k8s-app=kube-dns -o wide

Record the node where CoreDNS Pod is located.

---

### 16.2 Check resolv.conf on CoreDNS Node

Log in to the node where CoreDNS is located and execute:

    cat /etc/resolv.conf

Focus on the nameserver.

If the nameserver is an unreachable address, CoreDNS will fail to resolve external domain names.

---

### 16.3 Specify Upstream DNS Directly

If the /etc/resolv.conf on the node is unstable, you can change CoreDNS forward to a specific upstream DNS.

Example:

    forward . 223.5.5.5 114.114.114.114

Modify the ConfigMap:

    kubectl -n kube-system edit cm coredns

Change:

    forward . /etc/resolv.conf

To:

    forward . 223.5.5.5 114.114.114.114

Restart CoreDNS:

    kubectl -n kube-system rollout restart deployment coredns

Check the status:

    kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide

Test again:

    kubectl exec -it dns-test -- nslookup www.baidu.com

Note:

    In production environments, it is recommended to use the company's internal DNS or a reliable internal DNS.
    Do not arbitrarily write public DNS unless it complies with the company's network policies.

---

## Seventeen. Check CoreDNS Logs

Check CoreDNS logs:

    kubectl -n kube-system logs deploy/coredns --tail=100

If there are multiple replicas:

    kubectl -n kube-system logs -l k8s-app=kube-dns --tail=100

Common errors:

    plugin/forward: no nameservers found
    read udp timeout
    i/o timeout
    no such host
    loop detected
    CoreDNS-1.XX
    failed to list *v1.Service
    failed to list *v1.Endpoints
    failed to list *v1.Namespace

Common directions:

    i/o timeout
        Upstream DNS is unreachable or network is down.

    no nameservers found
        /etc/resolv.conf has no valid nameserver.

    loop detected
        DNS forwarding has a loop.

    failed to list Service/Endpoints
        CoreDNS has abnormal access to Kubernetes API or RBAC issues.

---

## Eighteen. Troubleshoot CoreDNS Pod CrashLoopBackOff

Check the Pod:

    kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide

Check details:

    kubectl -n kube-system describe pod <coredns-pod-name>

Check logs:

    kubectl -n kube-system logs <coredns-pod-name>

Common causes:

    1. Corefile configuration error
    2. Forward configuration error
    3. Loop plugin detects DNS loop
    4. CoreDNS image anomaly
    5. Node CNI anomaly
    6. CoreDNS cannot access APIServer

If the anomaly occurs after modifying the ConfigMap, check recent changes:

    kubectl -n kube-system get cm coredns -o yaml

Restore from backup if necessary.

---

## Nineteen. Check CoreDNS Replica Count

Check the Deployment:

    kubectl -n kube-system get deploy coredns

Check replica count:

    kubectl -n kube-system describe deploy coredns

Small clusters generally recommend at least 2 replicas.

If there is only 1 replica, you can scale it up:

    kubectl -n kube-system scale deployment coredns --replicas=2

Check:

    kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide

Note:

    CoreDNS replicas should not be too few.
    If the cluster has many nodes and applications, adjust based on QPS and resource usage.

---

## Twenty. Troubleshoot CoreDNS Resource Insufficiency

Check CoreDNS resources:

    kubectl -n kube-system get deploy coredns -o yaml | grep -A20 resources

Check real-time usage:

    kubectl top pod -n kube-system | grep coredns

If CoreDNS CPU or memory is consistently high, it may cause slow resolution or timeouts.

Common causes:

    1. High DNS query volume
    2. Applications frequently make short connections and repeat resolution
    3. ndots configuration leads to excessive queries
    4. Slow upstream DNS
    5. Insufficient CoreDNS replicas
    6. Unreasonable cache configuration

Handling directions:

    1. Increase CoreDNS replicas
    2. Appropriately increase resource requests/limits
    3. Check if the business has frequent DNS queries
    4. Optimize upstream DNS
    5. Keep the cache plugin enabled

---

## Twenty-one. Troubleshoot Pod dnsPolicy

Pods typically use:

    dnsPolicy: ClusterFirst

Check the Pod:

    kubectl get pod <pod-name> -n <namespace> -o yaml | grep dnsPolicy

Common types:

    ClusterFirst
    Default
    None
    ClusterFirstWithHostNet

Note:

ClusterFirst  
Default strategy, prioritizes cluster DNS.

Default  
Uses node DNS configuration, does not use Kubernetes cluster DNS.

None  
Custom dnsConfig.

ClusterFirstWithHostNet  
Use this strategy if hostNetwork Pods also need to use cluster DNS.

If business Pods use:

    dnsPolicy: Default

They may not resolve Kubernetes Service domain names.

---

## Twenty-two, DNS Issues with hostNetwork Pods

If a Pod uses:

    hostNetwork: true

DNS behavior may differ from normal Pods by default.

Check:

    kubectl get pod <pod-name> -n <namespace> -o yaml | grep -E "hostNetwork|dnsPolicy"

If a hostNetwork Pod needs to resolve cluster Services, recommend configuring:

    dnsPolicy: ClusterFirstWithHostNet

Example:

    spec:
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet

Common scenarios:

    1. Network plugin components
    2. Node-level DaemonSet
    3. ingress controller special deployment
    4. Monitoring components requiring host network

---

## Twenty-three, Troubleshooting Custom dnsConfig for Pods

Check if a Pod has configured dnsConfig:

    kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A20 dnsConfig

Example:

    dnsConfig:
      nameservers:
      - 8.8.8.8
      searches:
      - example.com
      options:
      - name: ndots
        value: "2"

If custom dnsConfig is configured incorrectly, it may cause Service domain name resolution issues.

Troubleshooting focus:

    1. Whether nameservers override cluster DNS
    2. Whether searches miss svc.cluster.local
    3. Whether ndots is unreasonable
    4. Whether dnsPolicy is None

---

## Twenty-four, ndots Causing Slow Resolution Issues

Pods often have /etc/resolv.conf with:

    options ndots:5

Meaning:

    Domains with fewer than 5 dots will prioritize appending search domains.

For example, accessing:

    www.baidu.com

May first try:

    www.baidu.com.default.svc.cluster.local
    www.baidu.com.svc.cluster.local
    www.baidu.com.cluster.local
    www.baidu.com

This may slow down external domain resolution.

Troubleshooting method:

    time nslookup www.baidu.com

    time nslookup www.baidu.com.

Note:

    Adding a dot at the end of a domain indicates an absolute domain:

        www.baidu.com.

Solutions:

    1. Use fully qualified domains with trailing dots in applications
    2. Optimize application DNS access methods
    3. Adjust Pod dnsConfig ndots if necessary
    4. Avoid arbitrary global changes to ndots

---

## Twenty-five, Troubleshooting Headless Service Resolution

Headless Services are commonly used with StatefulSet.

Check Service:

    kubectl get svc -n mysql

If it's a Headless Service:

    CLUSTER-IP: None

Example of full domain:

    mysql-0.mysql-headless.mysql.svc.cluster.local

Resolution test:

    nslookup mysql-headless.mysql.svc.cluster.local

    nslookup mysql-0.mysql-headless.mysql.svc.cluster.local

Common issues:

    1. Service is not Headless
    2. serviceName and StatefulSet are inconsistent
    3. Pod is not Ready
    4. publishNotReadyAddresses is not configured
    5. namespace is incorrect
    6. StatefulSet Pod name is incorrect

Check StatefulSet:

    kubectl get sts -n mysql

    kubectl get sts <sts-name> -n mysql -o yaml | grep serviceName

Check Pod:

    kubectl get pod -n mysql -o wide

Check Endpoints:

    kubectl get endpoints -n mysql

If StatefulSet needs to resolve unready Pods during startup, consider:

    publishNotReadyAddresses: true

But understand business startup logic before production use.

---

## Twenty-six, CoreDNS Accessing APIServer Abnormalities

CoreDNS needs to access Kubernetes API to retrieve Service, Endpoint, Namespace resources.

If CoreDNS cannot access APIServer, internal Service resolution may fail.

Check CoreDNS logs:

    kubectl -n kube-system logs -l k8s-app=kube-dns --tail=100

May see:

    failed to list *v1.Service
    failed to list *v1.Endpoints
    failed to list *v1.Namespace

Troubleshoot:

    kubectl -n kube-system get sa coredns

    kubectl get clusterrole system:coredns

kubectl get clusterrolebinding system:coredns

Check CoreDNS Pod connectivity to APIServer:

    kubectl -n kube-system exec -it <coredns-pod-name> -- sh

If tools are available in the container, you can test:

    nslookup kubernetes.default

If the image lacks tools, you can use a temporary Pod for testing.

---

## 27. CoreDNS and NetworkPolicy

If the cluster has NetworkPolicy enabled, it may block:

    1. Business Pod to CoreDNS on 53/UDP
    2. Business Pod to CoreDNS on 53/TCP
    3. CoreDNS to upstream DNS
    4. CoreDNS to APIServer

Check NetworkPolicy:

    kubectl get networkpolicy -A

If the namespace has a default deny egress policy, you need to explicitly allow access to kube-dns.

Common allowed directions:

    UDP 53 to kube-system/kube-dns
    TCP 53 to kube-system/kube-dns
    CoreDNS to upstream DNS
    CoreDNS to APIServer 443

Notes:

    Do not arbitrarily delete NetworkPolicy in production environments.
    Should combine with security policies for precise allow rules.

---

## 28. CoreDNS and kube-proxy

Pod access to kube-dns Service fundamentally still relies on Service forwarding capabilities.

If kube-proxy is abnormal, Pods may be unable to access kube-dns ClusterIP.

Check kube-dns Service:

    kubectl -n kube-system get svc kube-dns

Check kube-proxy:

    kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide

Check kube-proxy configuration:

    kubectl -n kube-system get cm kube-proxy -o jsonpath='{.data.config\.conf}' | grep -E "mode:|scheduler:"

If using IPVS, check on the node:

    sudo ipvsadm -Ln | grep 10.96.0.10

Replace 10.96.0.10 with the actual kube-dns Service ClusterIP.

If there are no kube-dns Service rules in IPVS, it may indicate kube-proxy is abnormal.

---

## 29. DNS Unreachable but ClusterIP Access is Normal

If:

    wget http://nginx-demo.default.svc.cluster.local fails
    wget http://10.96.100.20 succeeds

It indicates the business Service itself may be normal, with the issue concentrated on DNS resolution.

Troubleshooting directions:

    1. Pod /etc/resolv.conf
    2. kube-dns Service
    3. CoreDNS Pod
    4. CoreDNS ConfigMap
    5. NetworkPolicy blocking 53
    6. kube-proxy forwarding kube-dns Service

---

## 30. DNS is Normal but Business Still Unreachable

If:

    nslookup nginx-demo.default.svc.cluster.local succeeds
    wget nginx-demo.default.svc.cluster.local fails

It indicates DNS resolution is not the main issue.

Continue troubleshooting:

    1. Service Endpoints being empty
    2. Service targetPort being correct
    3. Pod actually listening on the port
    4. Application listening on 0.0.0.0
    5. kube-proxy being normal
    6. NetworkPolicy blocking business ports

Reference:

    03-Service Normal but Business Unreachable: selector, Endpoints, ports, and kube-proxy troubleshooting.md

---

## 31. Common Issues Quick Check

| Phenomenon | Common Causes | Priority Check |
|---|---|---|
| nslookup kubernetes.default fails | CoreDNS base failure | coredns Pod / kube-dns Service |
| Full domain name of Service unreachable | Service does not exist or namespace error | kubectl get svc |
| Short domain name unreachable, full domain name reachable | search domain or namespace issue | /etc/resolv.conf |
| External domain unreachable, internal domain reachable | upstream DNS issue | CoreDNS forward |
| CoreDNS CrashLoopBackOff | Corefile configuration error | coredns logs |
| Slow DNS resolution | ndots, upstream slow, CoreDNS pressure | resolv.conf / logs / top |
| Headless Service unreachable | StatefulSet / serviceName / Pod Ready | sts / endpoints |
| DNS normal but business unreachable | Service backend issue | endpoints / targetPort |
| Pod using hostNetwork has resolution anomalies | dnsPolicy not suitable | ClusterFirstWithHostNet |
| Only certain namespace DNS unreachable | NetworkPolicy | networkpolicy |

---

## 32. Standard Troubleshooting Command List

### 32.1 CoreDNS Status

    kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide

    kubectl -n kube-system describe pod <coredns-pod-name>

    kubectl -n kube-system logs -l k8s-app=kube-dns --tail=100

---

### 32.2 kube-dns Service

    kubectl -n kube-system get svc kube-dns

    kubectl -n kube-system describe svc kube-dns

kubectl -n kube-system get endpoints kube-dns

kubectl -n kube-system get endpointslice -l kubernetes.io/service-name=kube-dns

---

### 32.3 CoreDNS Configuration

    kubectl -n kube-system get cm coredns

    kubectl -n kube-system get cm coredns -o yaml

    kubectl -n kube-system get cm coredns -o jsonpath='{.data.Corefile}'

---

### 32.4 Pod Internal Testing

    kubectl run dns-test --image=busybox:1.36 --restart=Never -- sleep 3600

    kubectl exec -it dns-test -- cat /etc/resolv.conf

    kubectl exec -it dns-test -- nslookup kubernetes.default

    kubectl exec -it dns-test -- nslookup kubernetes.default.svc.cluster.local

    kubectl exec -it dns-test -- nslookup <service>.<namespace>.svc.cluster.local

    kubectl exec -it dns-test -- nslookup www.baidu.com

---

### 32.5 Service and Endpoints

    kubectl get svc -A

    kubectl get endpoints -A

    kubectl get endpointslice -A

    kubectl describe svc <service-name> -n <namespace>

    kubectl get endpoints <service-name> -n <namespace>

---

### 32.6 kube-proxy

    kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide

    kubectl -n kube-system logs -l k8s-app=kube-proxy --tail=100

    kubectl -n kube-system get cm kube-proxy -o jsonpath='{.data.config\.conf}' | grep -E "mode:|scheduler:|strictARP:"

    sudo ipvsadm -Ln

---

## Thirty-Three, Recommended Troubleshooting Path

### 33.1 Service Domain Name Cannot Be Resolved Inside Pod

Execution order:

    kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide

    kubectl -n kube-system get svc kube-dns

    kubectl -n kube-system get endpoints kube-dns

    kubectl exec -it dns-test -- cat /etc/resolv.conf

    kubectl exec -it dns-test -- nslookup kubernetes.default

    kubectl get svc <service-name> -n <namespace>

    kubectl get endpoints <service-name> -n <namespace>

---

### 33.2 External Domain Cannot Be Resolved

Execution order:

    kubectl exec -it dns-test -- nslookup www.baidu.com

    kubectl -n kube-system get cm coredns -o jsonpath='{.data.Corefile}'

    kubectl -n kube-system logs -l k8s-app=kube-dns --tail=100

    Check /etc/resolv.conf of the node where CoreDNS resides

    Test reachability of upstream DNS

---

### 33.3 Headless Service Resolution Abnormality

Execution order:

    kubectl get svc -n <namespace>

    kubectl get sts -n <namespace>

    kubectl get pod -n <namespace> -o wide

    kubectl get endpoints -n <namespace>

    nslookup <pod-name>.<headless-service>.<namespace>.svc.cluster.local

---

## Thirty-Four, Handling Recommendations

CoreDNS Troubleshooting Recommendations:

    1. First test kubernetes.default
    2. Then test the full domain name of the business Service
    3. Then test external domains
    4. Do not change CoreDNS ConfigMap immediately
    5. Backup ConfigMap before modifying CoreDNS
    6. Restart CoreDNS after modifying upstream DNS
    7. Service domain resolution does not guarantee business connectivity
    8. After DNS is normal, still check Endpoints and ports
    9. CoreDNS anomalies may be affected by kube-proxy, CNI, NetworkPolicy
    10. Use company internal DNS for upstream DNS in production environments

Backup CoreDNS configuration:

    kubectl -n kube-system get cm coredns -o yaml > coredns-cm-backup.yaml

Restore:

    kubectl apply -f coredns-cm-backup.yaml

Restart:

    kubectl -n kube-system rollout restart deployment coredns

---

## Thirty-Five, Summary

CoreDNS is the core component for service discovery within Kubernetes clusters.

When troubleshooting DNS issues, first distinguish: /think

1. Is the Service domain within the cluster unreachable?
2. Or is the external domain unreachable?
3. Is DNS resolution failing?
4. Or is DNS resolution successful but business access failing?

Core Checkpoints:

    1. Is the CoreDNS Pod Running
    2. Does the kube-dns Service exist
    3. Is the kube-dns Endpoints empty
    4. Is the Pod /etc/resolv.conf correct
    5. Can kubernetes.default be resolved
    6. Does the business Service exist
    7. Is the namespace correct
    8. Is the upstream DNS of CoreDNS' forward available
    9. Are there timeout or loop logs in CoreDNS
    10. Can kube-proxy forward kube-dns Service

Most Important Commands:

    kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide

    kubectl -n kube-system get svc kube-dns

    kubectl -n kube-system get endpoints kube-dns

    kubectl exec -it dns-test -- cat /etc/resolv.conf

    kubectl exec -it dns-test -- nslookup kubernetes.default

Experience-Based Judgment:

    1. If kubernetes.default resolution fails, prioritize checking CoreDNS basic capabilities
    2. If only external domains fail, prioritize checking CoreDNS forward and upstream DNS
    3. If DNS resolution succeeds but access fails, prioritize checking Service, Endpoints, targetPort
    4. If Headless Service is abnormal, prioritize checking StatefulSet, serviceName, Endpoints
    5. Before modifying CoreDNS, must back up ConfigMap