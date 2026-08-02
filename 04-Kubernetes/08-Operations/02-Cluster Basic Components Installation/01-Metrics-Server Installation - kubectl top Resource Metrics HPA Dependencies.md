# 01-metrics-server Installation: kubectl top, Resource Metrics, and HPA Basic Dependencies

Recommended Path:

    04-Kubernetes/08-Operations/02-Cluster Base Component Installation/01-metrics-server Installation: kubectl top, Resource Metrics, and HPA Basic Dependencies.md

Tags:

    #Kubernetes
    #metrics-server
    #kubectl-top
    #HPA
    #ResourceIndicators
    #ClusterBasicComponents
    #TransportSurveillance

---

## I. Document Description

This document records the installation and verification methods of metrics-server in a Kubernetes cluster.

metrics-server is a commonly used resource metric collection component in Kubernetes clusters, mainly used to collect CPU and memory usage of nodes and Pods.

After installing metrics-server, you can use:

    kubectl top nodes
    kubectl top pods
    kubectl top pods -A

HPA also depends on the resource metrics provided by metrics-server.

Document Objectives:

    1. Install metrics-server
    2. Verify kubectl top node
    3. Verify kubectl top pod
    4. Understand the relationship between metrics-server and HPA
    5. Master common troubleshooting methods

Execution Node:

    k8s-master-01

Prerequisites:

    1. Kubernetes cluster has been deployed
    2. All Node statuses are Ready
    3. CoreDNS is normal
    4. CNI network is normal
    5. kubectl can access the cluster normally

---

## II. metrics-server Function

metrics-server is used to collect basic resource metrics in Kubernetes clusters.

Main collection targets:

    1. Node CPU usage
    2. Node memory usage
    3. Pod CPU usage
    4. Pod memory usage

Common uses:

    1. kubectl top node
    2. kubectl top pod
    3. HPA automatically scales based on CPU/Memory
    4. Simple view of cluster resource usage

Notes:

    metrics-server is not a complete monitoring system.
    metrics-server does not handle long-term storage of metrics.
    metrics-server is not suitable as a replacement for Prometheus.
    metrics-server is mainly used for real-time resource metric queries and HPA.

---

## III. Pre-deployment Checks

### 3.1 Check Node Status

Execute:

    kubectl get nodes -o wide

Expected:

    All Master and Worker nodes are Ready.

---

### 3.2 Check kube-system Components

Execute:

    kubectl -n kube-system get pods -o wide

Key confirmations:

    coredns is normal
    kube-proxy is normal
    calico-related components are normal
    kube-apiserver is normal
    kube-controller-manager is normal
    kube-scheduler is normal

---

### 3.3 Check APIService

Execute:

    kubectl get apiservice

At this point, it may not have:

    v1beta1.metrics.k8s.io

This is normal, and it will appear after installing metrics-server.

---

## IV. Install metrics-server

This document uses the official YAML installation method.

Note:

    If the current environment cannot access registry.k8s.io, it is recommended to synchronize the metrics-server image to the internal Harbor in advance, or modify the image address in the YAML file to a domestic accessible image address.

---

### 4.1 Create Directory

Execute:

    mkdir -p /root/k8s-yaml/metrics-server

    cd /root/k8s-yaml/metrics-server

---

### 4.2 Download metrics-server YAML

Execute:

    curl -LO https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

If GitHub download is slow, you can download it locally via browser and upload to:

    /root/k8s-yaml/metrics-server/components.yaml

Check the file:

    ls -lh components.yaml

Check the image address:

    grep -n "image:" components.yaml

---

### 4.3 Modify metrics-server Startup Parameters

In kubeadm self-built clusters, internal network clusters, or test environments, metrics-server may encounter certificate validation issues when accessing kubelet.

Common error messages are similar to:

    x509: cannot validate certificate because it doesn't contain any IP SANs

Therefore, it is usually necessary to add the parameter:

    --kubelet-insecure-tls

It is also recommended to specify the use of node InternalIP as priority:

    --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname

Before modification, back up:

    cp components.yaml components.yaml.bak.$(date +%F-%H%M%S)

Edit the file:

    vim components.yaml

Find the metrics-server container's args section, adjust to something like:

    args:
      - --cert-dir=/tmp
      - --secure-port=10250
      - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
      - --kubelet-use-node-status-port
      - --metric-resolution=15s
      - --kubelet-insecure-tls

If you don't want to edit manually, you can also use sed to insert after the specified parameter:

    sed -i.bak '/--kubelet-use-node-status-port/a\        - --kubelet-insecure-tls' components.yaml

Check: /think

```markdown
grep -n "kubelet-insecure-tls" components.yaml

grep -n "kubelet-preferred-address-types" components.yaml

Explanation:

    --kubelet-insecure-tls
        Skip kubelet certificate validation, suitable for kubeadm self-hosted clusters or test environments.

    --kubelet-preferred-address-types
        Specify the preferred node address type for metrics-server to access kubelet.

    --metric-resolution=15s
        Metric collection interval.

Production Recommendations:

    If the company environment has already issued kubelet certificates in a standardized manner, prioritize using valid certificates.
    If it's a self-hosted cluster, experimental environment, or internal controlled environment, using --kubelet-insecure-tls is easier to implement.

---

### 4.4 Optional: Modify Image Address

Check the image:

    grep -n "image:" components.yaml

If the default image cannot be pulled, replace it with an internal Harbor image.

Example:

    sed -i.bak 's#registry.k8s.io/metrics-server/metrics-server:#harbor.example.com/k8s/metrics-server:#g' components.yaml

Note:

    The harbor.example.com/k8s/metrics-server above is just an example.
    The actual environment needs to be replaced with the company's internal Harbor address.

If the current environment can directly pull registry.k8s.io, there's no need to modify the image.

---

### 4.5 Install metrics-server

Execute:

    kubectl apply -f components.yaml

Check resources:

    kubectl -n kube-system get deploy metrics-server

    kubectl -n kube-system get pods -l k8s-app=metrics-server -o wide

    kubectl -n kube-system get svc metrics-server

---

## Five. Verify metrics-server

### 5.1 Check Pod Status

Execute:

    kubectl -n kube-system get pods -l k8s-app=metrics-server -o wide

Expected status:

    Running

If it's ContainerCreating, ImagePullBackOff, or CrashLoopBackOff, troubleshoot the image, parameters, certificates, or network issues first.

---

### 5.2 Check metrics-server Logs

Execute:

    kubectl -n kube-system logs deploy/metrics-server

If there are no obvious errors in the logs, continue verifying the APIService.

---

### 5.3 Check APIService

Execute:

    kubectl get apiservice | grep metrics

Expected to see:

    v1beta1.metrics.k8s.io

Check detailed status:

    kubectl describe apiservice v1beta1.metrics.k8s.io

Focus on:

    Status
    Conditions
    Message
    Reason

Normally, you should see Available as True.

---

### 5.4 Verify kubectl top nodes

After metrics-server is installed, it typically takes several seconds to a couple of minutes to collect metrics.

Execute:

    kubectl top nodes

Expected output similar to:

    NAME            CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
    k8s-master-01   200m         5%     1600Mi          40%
    k8s-master-02   180m         4%     1500Mi          38%
    k8s-master-03   190m         4%     1550Mi          39%
    k8s-worker-01   120m         3%     1100Mi          30%
    k8s-worker-02   130m         3%     1200Mi          32%

If you run it immediately after installation, you might see:

    error: Metrics API not available

Wait for a while and run it again.

---

### 5.5 Verify kubectl top pods

Check Pod metrics in the default namespace:

    kubectl top pods

Check Pod metrics across all namespaces:

    kubectl top pods -A

Check Pod metrics in the kube-system namespace:

    kubectl top pods -n kube-system

---

## Six. Deploy Test Application to Verify Metrics

### 6.1 Create Test Deployment

Execute:

    kubectl create deployment nginx-metrics-test --image=nginx:1.25

Scale up:

    kubectl scale deployment nginx-metrics-test --replicas=2

Check Pods:

    kubectl get pods -o wide

After waiting for Pods to be Running, execute:

    kubectl top pods

If you can see CPU/Memory metrics for nginx-metrics-test related Pods, it indicates metrics-server is working properly.

---

### 6.2 Clean Up Test Resources

Execute:

    kubectl delete deployment nginx-metrics-test

---

## Seven. Relationship Between metrics-server and HPA

HPA stands for Horizontal Pod Autoscaler, used to automatically adjust Pod replicas based on metrics.

Common scaling criteria:

    1. CPU usage rate
    2. Memory usage rate
    3. Custom metrics
    4. External metrics
```

metrics-server primarily provides basic resource metrics:

    CPU
    Memory

Therefore, the most common HPA CPU / Memory auto-scaling typically relies on metrics-server.

If metrics-server is not functioning properly, common impacts include:

    1. kubectl top node is unavailable
    2. kubectl top pod is unavailable
    3. HPA cannot retrieve CPU / Memory metrics
    4. HPA status shows unknown
    5. HPA cannot scale properly

When checking HPA, you may see:

    targets: <unknown>/80%

In this case, priority should be given to checking metrics-server.

---

## EightI don't know.HPA Simple Verification Example

Explanation:

    This section only performs basic verification to confirm whether metrics-server can support HPA.
    Full usage of HPA can be written as a separate document.

---

### 8.1 Create Test Application

Create Deployment:

    kubectl create deployment hpa-nginx --image=nginx:1.25

Set replica count:

    kubectl scale deployment hpa-nginx --replicas=1

Set CPU request for container:

    kubectl set resources deployment hpa-nginx --requests=cpu=100m

Check:

    kubectl get deployment hpa-nginx -o yaml | grep -A5 resources

---

### 8.2 Create HPA

Execute:

    kubectl autoscale deployment hpa-nginx --cpu-percent=50 --min=1 --max=5

Check HPA:

    kubectl get hpa

If metrics-server is functioning properly, after waiting for some time, HPA's TARGETS should not remain unknown long-term.

Check detailed information:

    kubectl describe hpa hpa-nginx

---

### 8.3 Clean Up Test Resources

Execute:

    kubectl delete hpa hpa-nginx

    kubectl delete deployment hpa-nginx

---

## NineI don't know.Common Issue Troubleshooting

### 9.1 kubectl top nodes reports Metrics API not available

Phenomenon:

    kubectl top nodes
    error: Metrics API not available

Troubleshoot:

    kubectl -n kube-system get pods -l k8s-app=metrics-server -o wide

    kubectl get apiservice | grep metrics

    kubectl describe apiservice v1beta1.metrics.k8s.io

    kubectl -n kube-system logs deploy/metrics-server

Common causes:

    1. metrics-server Pod is not Running
    2. APIService is not Available
    3. metrics-server image pull failed
    4. metrics-server cannot access kubelet
    5. kubelet certificate verification failed
    6. Just installed, metrics not collected yet

---

### 9.2 metrics-server logs show x509 certificate error

Common error:

    x509: cannot validate certificate because it doesn't contain any IP SANs

Solution:

    Add the following to metrics-server startup parameters:

        --kubelet-insecure-tls

Check configuration:

    kubectl -n kube-system get deploy metrics-server -o yaml | grep kubelet-insecure-tls

Modify YAML and reapply:

    kubectl apply -f components.yaml

Wait for rebuild:

    kubectl -n kube-system rollout restart deployment metrics-server

---

### 9.3 metrics-server image pull failed

Check Pod:

    kubectl -n kube-system get pods -l k8s-app=metrics-server -o wide

Check detailed information:

    kubectl -n kube-system describe pod <metrics-server-pod-name>

Common causes:

    1. Node cannot access registry.k8s.io
    2. Image repository is restricted
    3. containerd has no proxy or image source configuration
    4. Internal Harbor has not synchronized this image

Solutions:

    1. Synchronize metrics-server image to internal Harbor
    2. Modify image address in components.yaml
    3. Reapply with kubectl apply -f components.yaml

---

### 9.4 HPA TARGETS is always unknown

Check HPA:

    kubectl get hpa

Check detailed information:

    kubectl describe hpa <hpa-name>

Check metrics-server:

    kubectl top nodes

    kubectl top pods -A

Common causes:

    1. metrics-server is not functioning properly
    2. Pod has no resources.requests.cpu set
    3. HPA target metric configuration is incorrect
    4. Business Pod has not generated metrics yet

Note:

    When HPA scales based on CPU usage, Pod must have CPU request configured.
    If no CPU request is set, HPA cannot calculate the percentage.

---

### 9.5 APIService is unavailable

Check:

    kubectl describe apiservice v1beta1.metrics.k8s.io

Focus on:

Conditions
Message
Reason

Check Service:

    kubectl -n kube-system get svc metrics-server

Check Endpoint:

    kubectl -n kube-system get endpoints metrics-server

If endpoints is empty, it indicates that the metrics-server Pod is not functioning as a backend.

---

## Ten. Uninstall metrics-server

If uninstallation is required:

    kubectl delete -f components.yaml

Check:

    kubectl -n kube-system get pods -l k8s-app=metrics-server

    kubectl get apiservice | grep metrics

If the APIService still remains, check:

    kubectl get apiservice v1beta1.metrics.k8s.io

Delete if necessary:

    kubectl delete apiservice v1beta1.metrics.k8s.io

---

## Eleven. Post-Installation Checklist

After installation, execute:

    kubectl -n kube-system get pods -l k8s-app=metrics-server -o wide

    kubectl get apiservice | grep metrics

    kubectl describe apiservice v1beta1.metrics.k8s.io

    kubectl top nodes

    kubectl top pods -A

Should meet:

    1. metrics-server Pod is Running
    2. v1beta1.metrics.k8s.io exists
    3. APIService Available is True
    4. kubectl top nodes outputs normally
    5. kubectl top pods -A outputs normally
    6. HPA can retrieve CPU/Memory metrics

---

## Twelve. Summary

metrics-server is one of the fundamental components of a Kubernetes cluster.

It primarily provides:

    1. Node CPU/Memory metrics
    2. Pod CPU/Memory metrics
    3. kubectl top query capability
    4. HPA source for basic resource metrics

Post-installation commands:

    kubectl top nodes

    kubectl top pods

    kubectl top pods -A

Troubleshooting focus:

    1. Whether metrics-server Pod is Running
    2. Whether APIService is Available
    3. Whether kubelet certificate access is normal
    4. Whether the image can be pulled
    5. Whether Pod has configured resources.requests