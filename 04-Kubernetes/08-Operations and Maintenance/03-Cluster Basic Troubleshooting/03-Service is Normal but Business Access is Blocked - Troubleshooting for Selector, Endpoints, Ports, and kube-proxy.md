# 03-Service is Normal but Business Access is Blocked: Troubleshooting for Selector, Endpoints, Ports, and kube-proxy

Recommended Path:

    04-Kubernetes/08-Ops/03-Cluster Basic Troubleshooting/03-Service is Normal but Business Access is Blocked: Troubleshooting for Selector, Endpoints, Ports, and kube-proxy.md

Tags:

    #Kubernetes
    #Service
    #Endpoints
    #EndpointSlice
    #kube-proxy
    #IPVS
    #Service Discovery
    #Business Access Troubleshooting
    #Cluster Basic Troubleshooting

---

## I. Document Description

This document records the troubleshooting methods when a Kubernetes Service appears normal, the Pod is also Running, but business access remains blocked.

This is a very common scenario in Kubernetes operations and maintenance.

Typical Phenomena:

    1. `kubectl get svc` returns normal results.
    2. `kubectl get pod` returns normal results.
    3. The Pod status is Running.
    4. The Service has a ClusterIP.
    5. However, access to the Service is blocked.
    6. The Ingress/Gateway backend returns 502 or 503 errors.
    7. Accessing the service name inside the Pod fails.
    8. `curl` requests to the Service IP show no response.
    9. External access through the exposed NodePort is blocked.

Objectives of This Document:

    1. Understand the relationship between Service, selector, Endpoints, and EndpointSlice.
    2. Determine whether a Service is actually connected to backend Pods.
    3. Check whether the selector matches the Pod labels.
    4. Verify whether the port, targetPort, and containerPort are consistent.
    5. Ensure that the business process inside the container is actually listening on the specified port.
    6. Investigate kube-proxy/IPVS forwarding rules.
    7. Check for issues with ClusterIP, NodePort, and Ingress backend connectivity.
    8. Establish a standard troubleshooting process.

Applicable Scenarios:

    1. Service is normal but access is blocked.
    2. Pod is Running but business access is blocked.
    3. Ingress access returns 502/503 errors.
    4. Gateway API backend returns 503 errors.
    5. NodePort access is blocked.
    6. Service Endpoints are empty.
    7. Incorrect configuration of Service targetPort.
    8. Abnormalities with kube-proxy.

---

## II. Service Access Chain

The typical access chain for a Kubernetes Service:

    Client
      |
      v
    Service ClusterIP:Port
      |
      v
    kube-proxy forwarding rules
      |
      v
    Endpoints/EndpointSlice
      |
      v
    PodIP:targetPort
      |
      v
    Business process inside the container

If using Ingress:

    Client
      |
      v
    Ingress Controller
      |
      v
    Service
      |
      v
    Endpoints/EndpointSlice
      |
      v
    PodIP:targetPort
      |
      v
    Business process inside the container

Key Points to Consider:

    The existence of a Service does not guarantee that business access will be possible.
    Just because a Pod is Running does not mean that the business process inside the container is listening on the specified port.
    Having a ClusterIP for a Service does not necessarily mean it has a backend.
    If Endpoints are empty, the Service cannot forward requests to any backend Pod.

---

## III. General Troubleshooting Approach

When a Service appears normal but business access is blocked, it is recommended to follow these steps in order:

    1. Verify whether the Service exists.
    2. Check the Service selector.
    3. Check the Pod labels.
    4. Confirm whether the selector matches the Pod labels.
    5. Verify whether Endpoints/EndpointSlice are empty.
    6. Check the Service port and targetPort settings.
    7. Ensure that the container inside the Pod is listening on the targetPort.
    8. Perform a `curl` request to the Service ClusterIP within the cluster.
    9. Directly perform a `curl` request to the Pod IP.
    10. Compare the results of the two requests to identify any differences.
    11. Check kube-proxy/IPVS configuration.
    12. Verify the NodePort/Ingress/Gateway entry layers.

Subordinate Troubleshooting Paths:

    Service access is blocked
        |
        |-- Endpoints are empty
        |       |
        |       |-- Selector and labels do not match
        |       |-- Pod is not Ready
        |       |-- Pod is in a different namespace
```bash
kubectl get ep nginx-demo -n default
``````markdown
wget -qO- http://10.96.100.20

Note:

If accessing the ClusterIP is successful but the domain name fails, focus on checking CoreDNS. If accessing the PodIP is successful but the ClusterIP fails, check Service/kube-proxy. If accessing the PodIP fails, examine the Pod application, ports, and CNI network.
---

## Section Sixteen: Comparing PodIP and ServiceIP

This is a crucial method for identifying the scope of the issue.

### 16.1 Direct Access to PodIP

To obtain the PodIP:

    kubectl get pod -n default -o wide

To access it:

    wget -qO- http://<PodIP>:<targetPort>

### 16.2 Accessing Service ClusterIP

To obtain the ClusterIP:

    kubectl get svc nginx-demo -n default

To access it:

    wget -qO- http://<ClusterIP>:<port>

### 16.3 Analyzing Results

| PodIP Access | ServiceIP Access | Issue |
|---|---|---|
| Failed | Failed | Backend Pod, application port, CNI |
| Successful | Failed | Service, Endpoints, kube-proxy |
| Successful | Successful | The Service connection is normal; continue checking Ingress/Gateway/external entrances |
| Failed | Successful | Rare; confirm the testing method and target port |
---

## Section Seventeen: Checking kube-proxy

The forwarding of a Service's ClusterIP depends on kube-proxy.

### 17.1 Viewing kube-proxy Pod

Execute:

    kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide

Ensure that kube-proxy is running normally on each node.

View logs:

    kubectl -n kube-system logs <kube-proxy-pod-name>

Or:

    kubectl -n kube-system logs -l k8s-app=kube-proxy --tail=100
---

### 17.2 Checking kube-proxy Mode

Execute:

    kubectl -n kube-system get cm kube-proxy -o jsonpath '{.data.config\.conf}' | grep -E "mode:|scheduler:|strictARP:"

Common modes include:

    iptables
    ipvs

For this series of clusters, it is recommended to use:

    mode: "ipvs"
    scheduler: "rr"

---

### 17.3 Checking IPVS Rules

If kube-proxy uses the ipvs mode, execute on the node:

    sudo ipvsadm -Ln

To view specific Service ClusterIP rules:

    sudo ipvsadm -Ln | grep <ClusterIP>

Example:

    sudo ipvsadm -Ln | grep 10.96.100.20

To view statistics:

    sudo ipvsadm -Ln --stats

If the Service exists but there are no corresponding IPVS rules, it may indicate an issue with kube-proxy.

---

### 17.4 Checking iptables Rules

If using the iptables mode, you can view rules using:

    sudo iptables-save | grep nginx-demo

Or:

    sudo iptables-save | grep <ClusterIP>

Note:

    The troubleshooting commands vary depending on the mode used.
    For IPVS mode, focus on ipvsadm.
    For iptables mode, focus on iptables-save.

---

## Section Eighteen: Troubleshooting NodePort Access Issues

If the Service type is NodePort:

    kubectl get svc -n default

Example:

    NAME         TYPE       CLUSTER-IP      PORT(S)
    nginx-demo   NodePort   10.96.100.20    80:30080/TCP

The access chain is:

    Client -> NodeIP:30080 -> Service -> Endpoints -> Pod

### 18.1 Checking NodePort

View the Service:

    kubectl describe svc nginx-demo -n default

Verify:

    NodePort
    Port
    TargetPort
    Endpoints

---

### 18.2 Checking Node Port

On the node, execute:

    sudo ss -lntp | grep 30080

Note:

    NodePort may not be listed as a regular process listening port.
    kube-proxy handles traffic through iptables/IPVS rules.
    The absence of a listening process in `ss` does not necessarily indicate an issue.

It is more recommended to check:

    sudo ipvsadm -Ln | grep 30080

---

### 18.3 Checking Firewalls and Security Groups

On the node, execute:

    sudo ufw status

    sudo iptables -L -n

In a cloud environment, also check:

    Security groups
    Firewalls
    Network ACLs
    External load balancing rules

---

### 18.4 Checking externalTrafficPolicy

View:

    kubectl get svc nginxView Service:

    Use the command `kubectl get svc <svc-name> -n <namespace> -o yaml` to view details.

Verify:

    Check whether the protocol is set to TCP or UDP.

Common Errors:

    1. DNS-based services use UDP, but the Service is configured with TCP.
    2. Incorrect protocol configuration for games, audio/video, or gateway services.
    3. Applications using gRPC are incorrectly configured with HTTP for Ingress/Gateway settings.
    4. HTTPS backends are being forwarded as HTTP.

---

## Issue 23: TargetPort Naming

The targetPort of a Service can be specified as either a number or a name.

Example:

    In the service configuration:
        ports:
            - port: 80
              targetPort: http
This requires that the Pod's containerPort has a name set to "http".

Pod Example:

    In the Pod configuration:
        ports:
            - name: http
              containerPort: 8080

If the Service's targetPort is specified as "http" but the containerPort does not have a name set to "http", it may cause port resolution issues for Endpoints.

Verification:

    Use the commands `kubectl get svc <svc-name> -n <namespace> -o yaml` and `kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A10 ports` to check the settings.

Recommendations:

    1. For simple services, use a numerical targetPort.
    2. For multi-port services, use named ports.
    3. Ensure that the names used for ports are exactly consistent.

---

## Issue 24: Multiple Ports on a Service

For Services with multiple ports, each port must be assigned a name.

Example:

    In the service configuration:
        ports:
            - name: http
              port: 80
              targetPort: 8080
            - name: metrics
              port: 9100
              targetPort: 9100

Common Issues:

    1. Missing port names.
    2. Incorrect targetPort values.
    3. Ingress backend referencing the wrong ports.
    4. Prometheus ServiceMonitor referencing the wrong ports.
    5. Gateway HTTPRoute backendRefs specifying incorrect ports.

Troubleshooting:

    Use the commands `kubectl get svc <svc-name> -n <namespace> -o yaml` and `kubectl describe svc <svc-name> -n <namespace>` to check the settings.

---

## Issue 25: Quick Reference for Common Problems

| Problem | Possible Causes | Priority Check Items |
|---|---|---|
| Service has ClusterIP but is unreachable | No Endpoints configured | Use `kubectl get endpoints` |
| No Endpoints | Selector and label mismatch | Verify `svc selector` and `pod labels` |
| Pod is running but not listed in Endpoints | ReadinessProbe failed | Check `pod READY` status and `describe pod` |
| PodIP is unreachable | Application is not listening, or CNI issues exist | Use `curl PodIP` or inspect container ports |
| PodIP is reachable but ServiceIP is unreachable | kube-proxy malfunction | Check `kube-proxy` and `ipvsadm` |
| Ingress returns 503 | No Endpoints configured for the Service | Verify `describe ingress` and `svc endpoints` |
| Ingress returns 502 | Incorrect backend port or connection failure | Check `targetPort` and Pod listening settings |
| NodePort is unreachable | Firewall, kube-proxy, or externalTrafficPolicy issues | Use `ipvsadm`, `ufw`, and check Pod distribution |
| Domain name is unreachable but ClusterIP is reachable | DNS problems | Verify CoreDNS configuration |
| Service targetPort is a named port | ContainerPort name does not match | Check `svc yaml` and `pod yaml` |

---

## Issue 26: Standard Troubleshooting Commands

### 26.1 Service Layer

    Use the commands:
        `kubectl get svc -n <namespace>`
        `kubectl describe svc <svc-name> -n <namespace>`
        `kubectl get svc <svc-name> -n <namespace> -o yaml`

---

### 26.2 Endpoints Layer

    Use the commands:
        `kubectl get endpoints <svc-name> -n <namespace>`
        `kubectl describe endpoints <svc-name> -n <namespace>`
        `kubectl get endpointslice -n <namespace> -l kubernetes.io/service-name=<svc-name>`
        `kubectl describe endpointslice -n <namespace> -l kubernetes.io/service-name=<svc-name>`

---

### 26.3 Pod Layer

    Use the commands:
        `kubectl get pods -n <namespace> -o wide --show-labels`
        `kubectl describe pod <pod-name> -n <namespace>`
        `kubectl    kubectl get endpoints <svc-name> -n <namespace>

    kubectl get pods -n <namespace> -o wide

    sudo ipvsadm -Ln | grep <NodePort>

    sudo ufw status

    curl http://<NodeIP>:<NodePort>

---

## Summary

When a Service is present but the service is not functioning, it is not enough to simply check whether the Service exists or whether the Pods are Running.

Key points to consider:

    1. Whether the Service selector is correct.
    2. Whether the Pod labels match.
    3. Whether the Endpoints are empty.
    4. Whether the EndpointSlice is functioning properly.
    5. Whether the targetPort is correct.
    6. Whether the container is actually listening on that port.
    7. Whether the application is listening on 0.0.0.0.
    8. Whether the PodIP can be accessed.
    9. Whether the ClusterIP can be accessed.
    10. Whether kube-proxy is working correctly.
    11. Whether IPVS/iptables rules are in place.
    12. Whether Ingress/Gateway is referencing the correct Service and port.

Most important commands:

    kubectl describe svc <svc-name> -n <namespace>

    kubectl get endpoints <svc-name> -n <namespace>

    kubectl get pods -n <namespace> --show-labels

Tips for troubleshooting:

    1. If Endpoints are empty, check the selector, labels, and ReadinessProbe first.
    2. If PodIP is unreachable, check whether the application is listening on that port, the container's port settings, and CNI configurations.
    3. If PodIP is reachable but ServiceIP is not, check kube-proxy.
    4. If ServiceIP is reachable but Ingress is not, examine Ingress rules and Controller logs.
    5. If NodePort is unreachable, check the firewall, kube-proxy, and externalTrafficPolicy settings.
    6. Do not assume that just because a Service exists, it will be accessible.