# 04-CoreDNS Troubleshooting: Service Domain Name Resolution, Upstream DNS, and nslookup within Pods

Recommended Path:

    04-Kubernetes/08-Operations/03-Cluster Basic Troubleshooting/04-CoreDNS Troubleshooting: Service Domain Name Resolution, Upstream DNS, and nslookup within Pods.md

Tags:

    #Kubernetes
    #CoreDNS
    #DNS
    #Service Discovery
    #Service Resolution
    #Pod Networking
    #Cluster Basic Troubleshooting

---

## I. Document Overview

This document outlines basic troubleshooting methods for CoreDNS-related issues in Kubernetes clusters.

CoreDNS is the internal DNS component of a Kubernetes cluster, responsible for:

    1. Resolving Service domain names within pods
    2. Resolving Headless Service domain names within pods
    3. Resolving external domain names within pods
    4. Facilitating service discovery in Kubernetes
    5. Forwarding external domain names to the upstream DNS server

Common Issues:

    1. Inability to resolve Service domain names within pods
    2. Inability to resolve external domain names within pods
    3. Failure of `nslookup kubernetes.default`
    4. Services appearing normal but domain name access failing
    5. CoreDNS Pod crashing in a LoopBackOff state
    6. Timeout errors in CoreDNS logs
    7. Incorrect configuration of the upstream DNS server
    8. Abnormal domain name resolution for StatefulSet Headless Services
    9. Issues with resolving backend service names through Ingress/Gateway

Objectives:

    1. Understand the relationship between CoreDNS, Services, and Pod-level DNS
    2. Master methods for verifying DNS functionality within pods
    3. Learn how to monitor the status of CoreDNS Pods
    4. Know how to inspect the `kube-dns` Service
    5. Understand how to check the CoreDNS ConfigMap
    6. Be able to troubleshoot issues with Service domain name resolution
    7. Diagnose failures in external domain name resolution
    8. Resolve problems related to Headless Service resolution
    9. Establish a standard process for DNS troubleshooting

Applicable Scenarios:

    1. Self-built Kubernetes clusters using kubeadm
    2. Privately managed Kubernetes clusters
    3. Cases where service domain name access within pods fails
    4. Situations where external domain name access within pods fails
    5. CoreDNS-related anomalies
    6. Service discovery failures
    7. Abnormalities in StatefulSet domain name resolution

---

## II. Role of CoreDNS in Kubernetes

Common internal domain name formats in Kubernetes:

    <service-name>.<namespace>.svc.cluster.local

Examples:

    kubernetes.default.svc.cluster.local
    nginx-demo.default.svc.cluster.local
    mysql.mysql.svc.cluster.local

When accessing Services within pods, the following names are typically used:

    nginx-demo
    nginx-demo.default
    nginx-demo.default.svc
    nginx-demo.default.svc.cluster.local

CoreDNS is responsible for resolving these domain names into the Service's ClusterIP or the backend Pod's IP address.

---

## III. DNS Access Mechanism

When a pod accesses a Service domain name, the process generally follows this path:

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
    Kubernetes API/Service records
     |
     v
    Returns the Service's ClusterIP

For accessing external domain names, the process is similar but involves an additional step with the CoreDNS forward plugin:

    Pod
     |
     v
    /etc/resolv.conf
     |
     v
    kube-dns Service ClusterIP
     |
     v
    CoreDNS forward plugin
     |
     v
    Upstream DNS server
     |
     v
    Returns the external domain name resolution result

---

## IV. Relationship between CoreDNS and `kube-dns` Service

In Kubernetes, the Service is typically named:

    kube-dns

However, the actual Pod responsible for handling DNS requests is usually named:

    CoreDNS

To inspect the `kube-dns` Service:

    kubectl -n kube-system get svc kube-dns

Example Output:

    NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)
    kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP

Explanation:

    `kube-dns` is the Service name.
    The CoreDNS Pod is responsible for handling DNS requests.
    The `/etc/resolv1. Is the CoreDNS Pod running?
2. Does the CoreDNS Pod label match the Service selector?
3. Is the CoreDNS Pod Ready?
4. Are there any abnormalities with the kube-dns Service selector?

To check the Service selector:

    kubectl -n kube-system get svc kube-dns -o yaml | grep -A10 selector

To check the CoreDNS Pod label:

    kubectl -n kube-system get pods --show-labels | grep coredns

---

## Step 9: Create a DNS Test Pod

It is recommended to prepare a temporary Pod specifically for DNS troubleshooting.

Create it:

    kubectl run dns-test \
      --image=busybox:1.36 \
      --restart=Never \
      --sleep 3600

View it:

    kubectl get pod dns-test -o wide

Enter the Pod:

    kubectl exec -it dns-test -- sh

If the temporary Pod cannot be started, you need to troubleshoot first:

    1. Image pull
    2. Node status
    3. CNI
    4. Pod Pending

---

## Step 10: Check the resolv.conf inside the Pod

Execute the following command inside the dns-test Pod:

    cat /etc/resolv.conf

Normal example:

    nameserver 10.96.0.10
    search default.svc.cluster.local svc.cluster.local cluster.local
    options ndots:5

Key points to note:

    nameserver
        Should be the ClusterIP of the kube-dns Service.

    search
        Should include the current namespace and relevant search domains like cluster.local.

    ndots
        In Kubernetes, it is usually set to 5 by default.

If the nameserver is not the kube-dns ClusterIP, you need to check:

    1. kubelet DNS configuration
    2. Cluster DNS Service IP
    3. Pod dnsPolicy
    4. Pod dnsConfig

---

## Step 11: Test the resolution of Kubernetes default Services

Execute the following command inside the dns-test Pod:

    nslookup kubernetes.default

Or:

    nslookup kubernetes.default.svc.cluster.local

It should resolve to the Kubernetes Service ClusterIP.

View the Kubernetes Service:

    kubectl get svc kubernetes -n default

Example:

    NAME         TYPE        CLUSTER-IP   PORT(S)
    kubernetes   ClusterIP   10.96.0.1    443/TCP

If nslookup kubernetes.default fails, it indicates that there is an issue with the internal DNS infrastructure of the cluster.

---

## Step 12: Test the resolution of business Services

Assume the business Service is:

    namespace: default
    service: nginx-demo

View the Service:

    kubectl get svc nginx-demo -n default

Test short domain names:

    nslookup nginx-demo

Test with a namespace:

    nslookup nginx-demo.default

Test full domain names:

    nslookup nginx-demo.default.svc.cluster.local

If the full domain name can be resolved but the short domain name cannot, you need to check the namespace and search domain of the current Pod.

---

## Step 13: Troubleshoot Service domain name resolution failures

### 13.1 Check if the Service exists

Execute:

    kubectl get svc -n default

View the specified Service:

    kubectl get svc nginx-demo -n default

If the Service does not exist, DNS will definitely not be able to resolve that Service's domain name.

---

### 13.2 Check if the namespace is correct

Service domain names include the namespace.

For example, if the Service is in app-prod:

    nginx-demo.app-prod.svc.cluster.local

If accessed from a Pod in the default namespace:

    nginx-demo

It will be resolved as:

    nginx-demo.default.svc.cluster.local

This will result in a failure. The correct way to access it is:

    nginx-demo.app-prod

Or:

    nginx-demo.app-prod.svc.cluster.local

---

### 13.3 Check the Service type

For regular ClusterIP Services:

    nslookup <service>.<namespace>.svc.cluster.local

It usually resolves to the ClusterIP.

For Headless Services:

    clusterIP: None

It usually resolves to a list of backend Pod IPs.

View it:

    kubectl get svc <service-name> -n <namespace> -o yaml | grep clusterIP

---

### 13.4 Check the Endpoints

Just because DNS can resolve a Service does not mean that the service is accessible. If the Service is a regular ClusterIP, DNS resolution usually returns the ClusterIP. However, if the Endpoints are empty, access will still be unavailable.

Check:

    kubectl get endpoints <service-name> -n <namespace>

    kubectl get endpointslice -n <namespace> -Record the node where the CoreDNS Pod is located.

---

### 16.2 Viewing resolv.conf on the Node Where CoreDNS Resides

Log in to the node where CoreDNS resides and execute:

    cat /etc/resolv.conf

Pay special attention to the nameserver entries.

If the nameserver is an unreachable address, CoreDNS will fail to resolve external domain names.

---

### 16.3 Directly Specifying an Upstream DNS Server

If the /etc/resolv.conf on the node is unstable, you can modify CoreDNS's forward settings to use a specific upstream DNS server.

For example:

    forward . 223.5.5.5 114.114.114.114

Modify the ConfigMap:

    kubectl -n kube-system edit cm coredns

Change the following line:

    forward . /etc/resolv.conf

to:

    forward . 223.5.5.5 114.114.114.114

Restart CoreDNS:

    kubectl -n kube-system rollout restart deployment coredns

Check the status:

    kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide

Test again:

    kubectl exec -it dns-test -- nslookup www.baidu.com

Note:

    In a production environment, it is recommended to use the company's internal DNS or a reliable private network DNS server.
    It is not advisable to randomly specify a public DNS server unless it complies with the company's network policies.

---

## Section Seventeen: Viewing CoreDNS Logs

To view CoreDNS logs:

    kubectl -n kube-system logs deploy/coredns --tail=100

If there are multiple replicas:

    kubectl -n kube-system logs -l k8s-app=kube-dns --tail=100

Common errors include:

    plugin/forward: no nameservers found
    read udp timeout
    i/o timeout
    no such host
    loop detected
    CoreDNS-1.XX
    failed to list *v1.Service
    failed to list *v1.Endpoints
    failed to list *v1.Namespace

Common causes include:

    i/o timeout
        The upstream DNS server is unreachable or the network is down.

    no nameservers found
        There are no valid nameserver entries in /etc/resolv.conf.

    loop detected
        A DNS forwarding loop has occurred.

    failed to list Service/Endpoints
        There may be an issue with CoreDNS accessing the Kubernetes API or RBAC permissions.

---

## Section Eighteen: Troubleshooting CoreDNS Pod CrashLoopBackOff

To view the Pod information:

    kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide

For more details:

    kubectl -n kube-system describe pod <coredns-pod-name>

To view the logs:

    kubectl -n kube-system logs <coredns-pod-name>

Common causes include:

    1. Incorrect Corefile configuration
    2. Errors in the forward settings
    3. The loop plugin detecting a DNS forwarding loop
    4. Issues with the CoreDNS image
    5. CNI anomalies on the node
    6. CoreDNS being unable to access the APIServer

If the issue occurs after modifying the ConfigMap, you can check the recent changes:

    kubectl -n kube-system get cm coredns -o yaml

 Restore the backup configuration if necessary.

---

## Section Nineteen: Checking the Number of CoreDNS Replicas

To view the Deployment information:

    kubectl -n kube-system get deploy coredns

To check the number of replicas:

    kubectl -n kube-system describe deploy coredns

For small clusters, it is generally recommended to have at least 2 replicas.

If there is only 1 replica, you can scale up the deployment:

    kubectl -n kube-system scale deployment coredns --replicas=2

To verify the changes:

    kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide

Note:

    It is not advisable to have too few CoreDNS replicas.
    If the cluster has many nodes and multiple services, you may need to adjust the number of replicas based on QPS and resource usage.

---

## Section Twenty: Troubleshooting Insufficient CoreDNS Resources

To view CoreDNS resource information:

    kubectl -n kube-system get deploy coredns -o yaml | grep -A20 resources

To check real-time resource usage:

    kubectl top pod -n kube-system | grep coredns

If CoreDNS consumes high amounts of CPU or memory for an extended period, it may cause slow resolution or timeouts.

Common causes include:

    1. A largetime nslookup www.baidu.com

time nslookup www.baidu.com.

Note:

Adding a period at the end of a domain name indicates an absolute domain name:

www.baidu.com.

Treatment methods:

1. Use the full domain name with a period at the end in applications.
2. Optimize the DNS access method in applications.
3. Adjust the Pod's dnsConfig ndots if necessary.
4. It is not recommended to randomly modify ndots globally.

---

## 25. Troubleshooting for Headless Service Resolution

Headless Services are often used with StatefulSets.

View the Service:

kubectl get svc -n mysql

If it is a Headless Service:

CLUSTER-IP: None

Example of a full domain name:

mysql-0.mysql-headless.mysql.svc.cluster.local

Resolution test:

nslookup mysql-headless.mysql.svc.cluster.local

nslookup mysql-0/mysql-headless.mysql.svc.cluster.local

Common issues:

1. The Service is not Headless.
2. The serviceName does not match the StatefulSet.
3. The Pod is not Ready.
4. publishNotReadyAddresses is not configured.
5. The namespace is misspelled.
6. The name of the StatefulSet Pod is misspelled.

View the StatefulSet:

kubectl get sts -n mysql

kubectl get sts <sts-name> -n mysql -o yaml | grep serviceName

View the Pod:

kubectl get pod -n mysql -o wide

View the Endpoints:

kubectl get endpoints -n mysql

If a StatefulSet needs to allow unresolved Pods during startup, consider setting:

publishNotReadyAddresses: true

However, this should be carefully considered before production use based on the business logic.

---

## 26. Abnormalities When CoreDNS Attempts to Access the APIServer

CoreDNS needs to access the Kubernetes API to retrieve resources such as Services, Endpoints, and Namespaces.

If CoreDNS cannot access the APIServer, Service resolution within the cluster may be affected.

View CoreDNS logs:

kubectl -n kube-system logs -l k8s-app=kube-dns --tail=100

You may see messages like:

failed to list *v1.Service
failed to list *v1.Endpoints
failed to list *v1.Namespace

Troubleshooting steps:

kubectl -n kube-system get sa coredns

kubectl get clusterrole system:coredns

kubectl get clusterrolebinding system:coredns

Check the connectivity between the CoreDNS Pod and the APIServer:

kubectl -n kube-system exec -it <coredns-pod-name> -- sh

If tools are available in the container, you can test:

nslookup kubernetes.default

If the image lacks these tools, you can use a temporary Pod for testing.

---

## 27. CoreDNS and NetworkPolicy

If NetworkPolicy is enabled in the cluster, it may block:

1. The 53/UDP traffic from business Pods to CoreDNS.
2. The 53/TCP traffic from business Pods to CoreDNS.
3. Traffic from CoreDNS to upstream DNS servers.
4. Traffic from CoreDNS to the APIServer.

View NetworkPolicy settings:

kubectl get networkpolicy -A

If there is a default deny egress policy in the namespace, explicit allowlist rules are needed for kube-dns access.

Common allowed directions include:

UDP 53 to kube-system/kube-dns
TCP 53 to kube-system/kube-dns
CoreDNS to upstream DNS servers
CoreDNS to APIServer on port 443

Note:

Do not remove NetworkPolicy rules in a production environment. Always ensure they are configured in line with security policies.

---

## 28. CoreDNS and kube-proxy

When Pods access the kube-dns Service, it ultimately relies on the Service's forwarding capabilities.

If kube-proxy encounters an issue, Pods may not be able to reach the kube-dns ClusterIP.

View the kube-dns Service:

kubectl -n kube-system get svc kube-dns

View kube-proxy:

kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide

Check kube-proxy configuration:

kubectl -n kube-system get cm kube-proxy -o jsonpath '{.data.config\.conf}' | grep -E "mode:|scheduler:"

If IPVS is being used, check on the node:

sudo ipvsadm -Ln | grep 10.96.0.10

Replace 10.96.0.10 with the actual kube-dns Service ClusterIP.

If there are no rules for the kube-dns Service in IPVS, it may indicate an issue with kube-proxy.

---

## 29. DNS Resolution Fails but Direct Access to the ClusterIP Works

If:

wget http://nginx-demo.default.svc.cluster.local fails
wget http://10.96.100.20 works successfully

It suggests that### 32.2 kube-dns Service

    kubectl -n kube-system get svc kube-dns

    kubectl -n kube-system describe svc kube-dns

    kubectl -n kube-system get endpoints kube-dns

    kubectl -n kube-system get endpointslice -l kubernetes.io/service-name=kube-dns

---

### 32.3 CoreDNS Configuration

    kubectl -n kube-system get cm coredns

    kubectl -n kube-system get cm coredns -o yaml

    kubectl -n kube-system get cm coredns -o jsonpath '{.data.Corefile}'

---

### 32.4 Pod Internal Testing

    kubectl run dns-test --image=busybox:1.36 --restart=Never -- sleep 3600

    kubectl exec -it dns-test -- cat /etc/resolv.conf

    kubectl exec -it dns-test -- nslookup kubernetes.default

    kubectl exec -it dns-test -- nslookup kubernetes.default.svc.cluster.local

    kubectl exec -it dns-test -- nslookup <service>.<namespace>.svc.cluster.local

    kubectl exec -it dns-test -- nslookup www.baidu.com

---

### 32.5 Services and Endpoints

    kubectl get svc -A

    kubectl get endpoints -A

    kubectl get endpointslice -A

    kubectl describe svc <service-name> -n <namespace>

    kubectl get endpoints <service-name> -n <namespace>

---

### 32.6 kube-proxy

    kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide

    kubectl -n kube-system logs -l k8s-app=kube-proxy --tail=100

    kubectl -n kube-system get cm kube-proxy -o jsonpath '{.data.config\.conf}' | grep -E "mode:|scheduler:|strictARP:"

    sudo ipvsadm -Ln

---

## Section 33: Recommended Troubleshooting Paths

### 33.1 Inability to Resolve Service Domain Names Within Pods

Execution sequence:

    kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide

    kubectl -n kube-system get svc kube-dns

    kubectl -n kube-system get endpoints kube-dns

    kubectl exec -it dns-test -- cat /etc/resolv.conf

    kubectl exec -it dns-test -- nslookup kubernetes.default

    kubectl get svc <service-name> -n <namespace>

    kubectl get endpoints <service-name> -n <namespace>

---

### 33.2 Inability to Resolve External Domain Names

Execution sequence:

    kubectl exec -it dns-test -- nslookup www.baidu.com

    kubectl -n kube-system get cm coredns -o jsonpath '{.data.Corefile}'

    kubectl -n kube-system logs -l k8s-app=kube-dns --tail=100

    Check the /etc/resolv.conf file on the node where CoreDNS is running.

    Verify if the upstream DNS server is accessible.

---

### 33.3 Abnormal Resolution of Headless Services

Execution sequence:

    kubectl get svc -n <namespace>

    kubectl get sts -n <namespace>

    kubectl get pod -n <namespace> -o wide

    kubectl get endpoints -n <namespace>

    nslookup <pod-name>.<headless-service>.<namespace>.svc.cluster.local

---

## Section 34: Handling Recommendations

### CoreDNS Troubleshooting Suggestions:

    1. First, test the resolution of kubernetes.default.
    2. Then, test the full domain names of your service applications.
    3. Next, test external domain names.
    4. Do not modify the CoreDNS ConfigMap immediately.
    5. Always back up the ConfigMap before making any changes.
    6. Restart CoreDNS after updating the forward settings for upstream DNS.
    7. Just because a Service's domain name is resolved does not mean that the service is functioning correctly.
    8. Even if DNS resolution is normal, check endpoints and ports carefully.
    9. Issues with CoreDNS may be affected by kube-proxy, CNI, or NetworkPolicy.
    10. In a production environment, use the company's internal DNS for upstream connections.

### Backing Up CoreDNS Configuration:

    kubectl -n kube-system get cm coredns -o yaml > coredns-cm-backup.yaml

### Restoring Configuration:

    kubectl apply -f coredns-cm-backup.yaml

### Restarting CoreDNS:

    kubectl -n kube-system rollout restart deployment coredns

---

## Section 35: Summary

CoreDNS