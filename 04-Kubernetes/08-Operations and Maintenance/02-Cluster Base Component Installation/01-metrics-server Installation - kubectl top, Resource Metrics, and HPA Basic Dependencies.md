# 01-metrics-server Installation: kubectl top, Resource Metrics, and HPA Basic Dependencies

Recommended Path:

    04-Kubernetes/08-Operations/02-Cluster Basic Components Installation/01-metrics-server Installation: kubectl top, Resource Metrics, and HPA Basic Dependencies.md

Tags:

    #Kubernetes
    #metrics-server
    #kubectl-top
    #HPA
    #Resource Metrics
    #Cluster Basic Components
    #Operations Monitoring

---

## I. Document Description

This document records the installation and verification methods for metrics-server in a Kubernetes cluster.

Metrics-server is a commonly used resource metric collection component in Kubernetes clusters, primarily responsible for gathering CPU and memory usage data of nodes and pods.

After installing metrics-server, you can use the following commands:

    kubectl top nodes
    kubectl top pods
    kubectl top pods -A

Additionally, HPA relies on the resource metrics provided by metrics-server.

Objectives of this document:

    1. Install metrics-server
    2. Verify kubectl top node
    3. Verify kubectl top pod
    4. Understand the relationship between metrics-server and HPA
    5. Master common troubleshooting methods

Execution Node:

    k8s-master-01

Prerequisites:

    1. The Kubernetes cluster has been deployed.
    2. All nodes are in the Ready state.
    3. CoreDNS is functioning properly.
    4. CNI networking is operational.
    5. kubectl can access the cluster successfully.

---

## II. Role of Metrics-server

Metrics-server is used to collect basic resource metrics within a Kubernetes cluster.

Main collection targets:

    1. Node CPU usage
    2. Node memory usage
    3. Pod CPU usage
    4. Pod memory usage

Common uses:

    1. kubectl top node
    2. kubectl top pod
    3. HPA for automatic scaling based on CPU/memory usage
    4. Briefly viewing cluster resource utilization

Note:

    Metrics-server is not a complete monitoring system.
    Metrics-server does not handle long-term metric storage.
    Metrics-server is not intended to replace Prometheus.
    Metrics-server is mainly used for real-time resource metric queries and HPA.

---

## III. Pre-Deployment Checks

### 3.1 Check Node Status

Execution:

    kubectl get nodes -o wide

Expected result:

    All Master and Worker nodes should be in the Ready state.

---

### 3.2 Check kube-system Components

Execution:

    kubectl -n kube-system get pods -o wide

Key checks:

    coredns is functioning normally.
    kube-proxy is working properly.
    Calico-related components are healthy.
    kube-apiserver is running smoothly.
    kube-controller-manager is stable.
    kube-scheduler is operational.

---

### 3.3 Check APIService

Execution:

    kubectl get apiservice

At this point, the following service may not be available yet:

    v1beta1.metrics.k8s.io

This is normal; it will appear after installing metrics-server.

---

## IV. Install Metrics-server

This document uses the official YAML installation method.

Note:

    If your current environment cannot access registry.k8s.io, it is recommended to pre-download the metrics-server image to an internal Harbor or modify the image URL in the YAML file to a locally accessible domestic mirror address.

---

### 4.1 Create Directories

Execution:

    mkdir -p /root/k8s-yaml/metrics-server

    cd /root/k8s-yaml/metrics-server

---

### 4.2 Download Metrics-server YAML File

Execution:

    curl -LO https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

If the GitHub download is slow, you can download it locally and then copy it to:

    /root/k8s-yaml/metrics-server/components.yaml

Check the file:

    ls -lh components.yaml

View the image URL:

    grep -n "image:" components.yaml

---

### 4.3 Modify Metrics-server Startup Parameters

In self-built kubeadm clusters, internal networks, or testing environments, metrics-server may encounter certificate verification issues when accessing kubelet.

Common error messages include:

    x509: cannot validate certificate because it doesn't contain any IP SANs

To resolve this, you typically need to add the following parameter:

    --kubelet-insecure-tls

It is also recommended to specify that the node's InternalIP should be preferentially used:

    --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname

Before making changes, back up the file:

    cp components.yaml components.yaml.bak.$(date +%F-%H%M%S)

Edit the file using vim:

    vim components.yaml

Find the args section for the metrics-server container and adjust itIf there are no obvious errors in the logs, continue to verify the APIService.

---

### 5.3 Checking the APIService

Execute:

    kubectl get apiservice | grep metrics

Expected output:

    v1beta1.metrics.k8s.io

View detailed status:

    kubectl describe apiservice v1beta1.metrics.k8s.io

Pay attention to:

    Status
    Conditions
    Message
    Reason

Under normal circumstances, "Available" should be set to True.

---

### 5.4 Verifying kubectl top nodes

After installing the metrics-server, it usually takes several seconds to a minute for the metrics to be collected.

Execute:

    kubectl top nodes

Expected output similar to:

    NAME            CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
    k8s-master-01   200m         5%     1600Mi          40%
    k8s-master-02   180m         4%     1500Mi          38%
    k8s-master-03   190m         4%     1550Mi          39%
    k8s-worker-01   120m         3%     1100Mi          30%
    k8s-worker-02   130m         3%     1200Mi          32%

If you execute it immediately after installation, you may see:

    error: Metrics API not available

Wait for a while and then try again.

---

### 5.5 Verifying kubectl top pods

View Pod metrics in the default namespace:

    kubectl top pods

View Pod metrics in all namespaces:

    kubectl top pods -A

View Pod metrics in the kube-system namespace:

    kubectl top pods -n kube-system

---

## VI. Deploying a Test Application to Verify Metrics

### 6.1 Creating a Test Deployment

Execute:

    kubectl create deployment nginx-metrics-test --image=nginx:1.25

Scale up:

    kubectl scale deployment nginx-metrics-test --replicas=2

View Pods:

    kubectl get pods -o wide

After the Pod starts running, execute:

    kubectl top pods

If you can see CPU / Memory metrics for the nginx-metrics-test related Pods, it indicates that the metrics-server is working properly.

---

### 6.2 Cleaning Up Test Resources

Execute:

    kubectl delete deployment nginx-metrics-test

---

## VII. The Relationship Between metrics-server and HPA

HPA stands for Horizontal Pod Autoscaler, which automatically adjusts the number of Pod replicas based on metrics.

Common reasons for scaling include:

    1. CPU usage
    2. Memory usage
    3. Custom metrics
    4. External metrics

metrics-server primarily provides basic resource metrics:

    CPU
    Memory

Therefore, common HPA scaling operations for CPU / Memory usually rely on metrics-server.

If the metrics-server is not functioning properly, possible issues include:

    1. Inability to use kubectl top nodes
    2. Inability to use kubectl top pods
    3. HPA being unable to obtain CPU / Memory metrics
    4. The HPA status showing "unknown"
    5. HPA being unable to perform scaling properly

When checking the HPA, you may see:

    targets: <unknown>/80%

In such cases, it is necessary to check the metrics-server first.

---

## VIII. A Simple Example of Verifying HPA

Note:

    This section only serves as a basic verification to see if the metrics-server can support HPA.
    A detailed explanation of how to use HPA in its entirety could be covered in another article.

---

### 8.1 Creating a Test Application

Create a Deployment:

    kubectl create deployment hpa-nginx --image=nginx:1.25

Set the number of replicas:

    kubectl scale deployment hpa-nginx --replicas=1

Specify CPU requests for the container:

    kubectl set resources deployment hpa-nginx --requests=cpu=100m

View configuration:

    kubectl get deployment hpa-nginx -o yaml | grep -A5 resources

---

### 8.2 Creating an HPA

Execute:

    kubectl autoscale deployment hpa-nginx --cpu-percent=50 --min=1 --max=5

View the HPA settings:

    kubectl get hpa

If the metrics-server is working correctly, after waiting for a while, the HPA's TARGETS should no longer remain "unknown".

View detailed information:

    kubectl describe hpa hpa-nginx

---

### 8.3 Cleaning Up Test Resources

Execute:

    kubectlWhen HPA adjusts the scale based on CPU usage, the Pod must have a configured CPU request. Without a CPU request, HPA cannot calculate the percentage.

---

### 9.5 APIService Unavailable

To check:

    kubectl describe apiservice v1beta1.metrics.k8s.io

Pay attention to:

    Conditions
    Message
    Reason

Check the Service:

    kubectl -n kube-system get svc metrics-server

Check the Endpoint:

    kubectl -n kube-system get endpoints metrics-server

If the endpoints are empty, it means that the metrics-server Pod is not functioning properly as a backend.

---

## Ten: Uninstalling metrics-server

If you need to uninstall it:

    kubectl delete -f components.yaml

Verify:

    kubectl -n kube-system get pods -l k8s-app=metrics-server

    kubectl get apiservice | grep metrics

If the APIService is still present, check again:

    kubectl get apiservice v1beta1.metrics.k8s.io

Delete it if necessary:

    kubectl delete apiservice v1beta1.metrics.k8s.io

---

## Eleven: Post-Installation Verification Checklist

After installation, perform the following checks:

    kubectl -n kube-system get pods -l k8s-app=metrics-server -o wide

    kubectl get apiservice | grep metrics

    kubectl describe apiservice v1beta1.metrics.k8s.io

    kubectl top nodes

    kubectl top pods -A

The results should meet the following criteria:

    1. The metrics-server Pod is in the Running state.
    2. The v1beta1.metrics.k8s.io APIService exists.
    3. The "APIService Available" status is True.
    4. The `kubectl top nodes` command outputs correctly.
    5. The `kubectl top pods -A` command outputs correctly.
    6. HPA can retrieve CPU/Memory metrics.

---

## Twelve: Summary

metrics-server is a fundamental component of the Kubernetes cluster. It provides:

    1. Node-level CPU/Memory metrics.
    2. Pod-level CPU/Memory metrics.
    3. The ability to query these metrics using `kubectl top`.
    4. It serves as the source for basic resource metrics used by HPA.

Common commands after installation include:

    kubectl top nodes
    kubectl top pods
    kubectl top pods -A

Key troubleshooting points include:

    1. Ensuring the metrics-server Pod is running.
    2. Checking if the APIService is available.
    3. Verifying that kubelet can access the necessary certificates.
    4. Confirming that the required images can be pulled down.
    5. Checking if the Pod has proper resource requests configured.