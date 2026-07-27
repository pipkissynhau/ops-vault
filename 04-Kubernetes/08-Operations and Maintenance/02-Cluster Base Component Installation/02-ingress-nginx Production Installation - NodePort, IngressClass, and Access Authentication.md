# 02-ingress-nginx Production Installation: NodePort, IngressClass, and Access Authentication

Recommended Path:

    04-Kubernetes/08-Operations/02-Cluster Basic Components Installation/02-ingress-nginx Production Installation: NodePort, IngressClass, and Access Authentication.md

Tags:

    #Kubernetes
    #Ingress
    #ingress-nginx
    #IngressClass
    #NodePort
    #Layer 7 Gateway
    #Service Exposure
    #Cluster Basic Components

---

## I. Document Description

This document records the installation, verification, and troubleshooting methods for ingress-nginx in a Kubernetes cluster.

ingress-nginx is a common Layer 7 gateway controller in Kubernetes that forwards external HTTP/HTTPS requests to internal Services based on Ingress rules.

Deployment Method in This Document:

    Installation Method: Helm
    Controller: ingress-nginx
    Exposure Method: NodePort
    HTTP NodePort: 30080
    HTTPS NodePort: 30443
    IngressClass: nginx
    Number of Replicas: 2
    Suitable Environment: Kubeadm-built clusters, bare-metal clusters, private environments

Execution Node:

    k8s-master-01

Prerequisites:

    1. The Kubernetes cluster has been deployed.
    2. Nodes are in the Ready state.
    3. CNI is functioning correctly.
    4. CoreDNS is working properly.
    5. kube-proxy is operational.
    6. Helm is installed.
    7. kubectl can access the cluster without issues.

---

## II. The Relationship Between Ingress and ingress-nginx

Ingress is a Kubernetes resource object that only defines access rules.

ingress-nginx is the Ingress Controller responsible for actually listening to external traffic and forwarding it to the backend Services.

The relationship is as follows:

    User Request
       |
       v
    NodeIP:30080 / NodeIP:30443
       |
       v
    ingress-nginx-controller Service
       |
       v
    ingress-nginx-controller Pod
       |
       v
    Ingress Rules
       |
       v
    Service
       |
       v
    Pod

Note:

    Having only the Ingress YAML is not enough.
    An Ingress Controller is required for the Ingress rules to take effect.

---

## III. Gateway Address Planning

In this document, ingress-nginx is exposed using NodePort.

| Item | Planning |
|---|---|
| ingress-nginx Namespace | ingress-nginx |
| IngressClass | nginx |
| HTTP NodePort | 30080 |
| HTTPS NodePort | 30443 |
| Test Domain Name | demo.ops.local |
| Test Service | nginx-demo |
| Test Namespace | demo |

Examples of Access Methods:

    http://anyNodeIP:30080
    https://anyNodeIP:30443

For example:

    http://10.0.0.23:30080
    http://10.0.0.24:30080
    http://10.0.0.25:30080

Note:

    10.0.0.30 is the kube-vip address of the Kubernetes APIServer.
    It is not recommended to use the APIServer VIP directly as a business Ingress gateway.
    In production environments, it is advisable to plan separate load balancing solutions such as SLB, F5, Nginx, HAProxy, MetalLB, or dedicated business VIPs for the entrance.

---

## IV. Pre-Deployment Checks

### 4.1 Check Cluster Nodes

Execute:

    kubectl get nodes -o wide

Requirements:

    Master nodes must be in the Ready state.
    Worker nodes must also be in the Ready state.

---

### 4.2 Check Helm

Execute:

    helm version

If Helm is not installed, install it first.

---

### 4.3 Check kube-system Components

Execute:

    kubectl -n kube-system get pods -o wide

Focus on checking:

    coredns
    kube-proxy
    calico
    kube-apiserver
    kube-controller-manager
    kube-scheduler

---

### 4.4 Check NodePort Port Range

The default NodePort range in Kubernetes is:

    30000-32767

In this document, the following ports are used:

    30080
    30443

Check if these ports are already occupied:

    sudo ss -lntp | grep -E "30080|30443"

If no results are displayed, it means that no local process is using these ports.

---

## V### 14.1 Issues with the ingress-nginx-controller Pod's ImagePullBackOff

To diagnose this issue, first check the Pod details:

```bash
kubectl -n ingress-nginx get pods -o wide
```

Then examine the events associated with that Pod:

```bash
kubectl -n ingress-nginx describe pod <pod-name>
```

Common causes include:

1. Inability to access the registry.k8s.io.
2. Issues accessing the GitHub repository containing the image.
3. The image has not been synced to the internal Harbor.4. Containerd is unable to pull the image.

Solution:

1. Verify the node's network connectivity.
2. Check the image address.
3. Synchronize the image to the internal Harbor.
4. Modify the image configuration in the Helm values.
5. Reperform the helm upgrade.

To view the current default values of the Chart:

```
helm show values ingress-nginx/ingress-nginx > values-default.yaml
```

To check the image-related configurations:

```
grep -n "registry:" values-default.yaml
grep -n "image:" values-default.yaml
grep -n "tag:" values-default.yaml
```

---

### 14.2 Ingress access returns 404

Symptom:

The `curl` command returns a 404 Not Found error.

Troubleshooting:

```bash
kubectl -n demo get ingress
kubectl -n demo describe ingress nginx-demo
kubectl get ingressclass
```

Checkpoints:

1. Does the Host match?
2. Does the path match?
3. Is the `ingressClassName` set to "nginx"?
4. Does the request include the correct Host?
5. Is the Ingress being recognized by `ingress-nginx`?

Common error:

```
curl http://10.0.0.23:30080/
```

This request lacks the `Host: demo.ops.local`, which may prevent it from matching the Ingress rules.

Correct approach:

```
curl -H "Host: demo_ops.local" http://10.0.0.23:30080/
```

---

### 14.3 Ingress access returns 502/503 errors

Troubleshoot the Service:

```bash
kubectl -n demo get svc nginx-demo
```

Check the Endpoints:

```bash
kubectl -n demo get endpoints nginx-demo
```

Check the Pods:

```bash
kubectl -n demo get pods -o wide
kubectl -n demo describe pod <pod-name>
```

Common causes:

1. The `targetPort` of the Service is set incorrectly.
2. The Service selector does not match the Pod labels.
3. The Pod is not running.
4. The application inside the Pod is not listening on the correct port.
5. The Endpoints are empty.

---

### 14.4 NodePort access is unavailable

Check the Service:

```bash
kubectl -n ingress-nginx get svc ingress-nginx-controller
```

Check the NodePort:

```bash
kubectl -n ingress-nginx describe svc ingress-nginx-controller
```

Check the node port:

```bash
sudo ipvsadm -Ln | grep -E "30080|30443"
```

Check the kube-proxy:

```bash
kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide
```

Check the firewall:

```bash
sudo ufw status
```

Check port connectivity:

```bash
curl -I http://10.0.0.23:30080/
```

Common causes:

1. The NodePort is set incorrectly.
2. The security group or firewall does not allow access.
3. There is an issue with the kube-proxy.
4. The `ingress-nginx-controller` Pod is not running.
5. The `externalTrafficPolicy` configuration does not match the accessing node.

---

### 14.5 IngressClass does not match

Check the IngressClass:

```bash
kubectl get ingressclass
```

Check the Ingress:

```bash
kubectl -n demo get ingress nginx-demo -o yaml | grep ingressClassName
```

Requirement:

`ingressClassName` must be set to "nginx".

If the `ingressClassName` is not specified and the controller is configured with `watchIngressWithoutClass: false`, then the Ingress will not be processed.

Solution:

Clearly specify `ingressClassName` as "nginx":

```yaml
ingressClassName: nginx
```

---

### 14.6 Admission Webhook reports errors

Common symptoms:

- Failed to call the webhook.
- Validating the webhook failed.
- The certificate was signed by an unknown authority.

Troubleshooting:

```bash
kubectl -n ingress-nginx get jobs
kubectl -n ingress-nginx get pods | grep admission
kubectl -n ingress-nginx get validatingwebhookconfiguration
kubectl -n ingress-nginx describe pod <admission-pod-name>
```

Common causes:

1. The `admission Job` did not succeed.
2. The webhook certificate was not generated.
3. Image pull failed.
4. Installation was interrupted, leaving residual webhook components.

Solution:

1. Check the logs of the `admission Job`.
2. Reperform the helm upgrade.
3. If necessary, uninstall and reinstall the relevant components.
4. Verify if any residual `validating## XVIII. Summary

This document outlines the basic production-level installation of ingress-nginx.

Key points:

    1. Installed ingress-nginx using Helm.
    2. Exposed HTTP/HTTPS services via NodePort.
    3. Assigned fixed NodePort addresses: 30080 for HTTP and 30443 for HTTPS.
    4. Created an IngressClass named nginx.
    5. Developed a test application.
    6. Established Ingress rules.
    7. Implemented access validation using Host headers.
    8. Troubleshooted common issues such as 404 errors, 502/503 responses, unavailable NodePorts, and mismatched IngressClass configurations.

Further recommendations:

    03 - Installation of cert-manager: Automate TLS certificate issuance, renewal, and Secret management.md