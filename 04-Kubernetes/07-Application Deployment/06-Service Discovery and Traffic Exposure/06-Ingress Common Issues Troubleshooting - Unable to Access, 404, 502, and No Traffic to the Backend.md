# 06-Ingress Common Issues Troubleshooting: Unable to Access, 404, 502, and No Traffic to the Backend

## Document Description
- Document Location: This section serves as a summary for the Ingress learning phase within Kubernetes service discovery and traffic exposure.
- Applicable Stage: After understanding the role of Ingress, the basic principles of the Ingress Controller, and completing the first minimum practical example using kubeadm + ingress-nginx + NodePort, this document helps establish a troubleshooting approach for Ingress issues.
- Recommended Path: `04-Kubernetes/07-Application Deployment/06-Service Discovery and Traffic Exposure/06-Ingress Common Issues Troubleshooting: Unable to Access, 404, 502, and No Traffic to the Backend.md`

## Tags
#Kubernetes #Ingress #IngressController #ingress-nginx #Service #Endpoints #Pod #HTTP #404 #502 #Troubleshooting #kubeadm #NodePort #Application Deployment #Cloud-Native #Ops #Interview Notes

---

## I. Purpose of This Document

This document outlines the minimum troubleshooting steps for common Ingress issues in the current environment.

Prerequisites for the current environment include:

- Nodes:
  - `10.0.0.20`
  - `10.0.0.21`
  - `10.0.0.22`
- ingress-nginx Controller:
  - HTTP NodePort: `31152`
  - HTTPS NodePort: `30361`
- IngressClass:
  - `nginx -> k8s.io/ingress-nginx`

After completing this document, you should be able to:

- Troubleshoot Ingress issues along the actual request chain.
- Distinguish between "entry-level issues," "rule issues," and "backend issues."
- Understand the common causes of 404, 502, 503 errors, and no traffic to the backend.
- Master the most commonly used verification commands.

---

## II. General Principles for Ingress Troubleshooting

Ingress troubleshooting should not focus solely on a YAML file but should involve layer-by-layer checking along the actual traffic path.

The typical chain in the current environment is as follows:

    Client
    -> Domain name/hosts
    -> 10.0.0.20:31152 / 10.0.0.21:31152 / 10.0.0.22:31152
    -> ingress-nginx Controller
    -> Ingress rule matching
    -> Service
    -> Endpoints
    > Pod
    > Container process

Any issue at any layer of this chain can result in abnormal access to the entry point.

---

## III. Fixed Order of Troubleshooting

It is recommended to follow the order below for troubleshooting:

### Step 1: Verify if the request reaches the correct entry point
Confirm:

- Whether the hosts or domain name resolve to the correct node IP.
- Whether the node IP is reachable.
- Whether the current access port is `31152` or `30361`.

### Step 2: Check if the Controller is functioning normally
Confirm:

- Whether the ingress-nginx Controller Pod is `Running`.
- Whether the `ingress-nginx-controller` Service exists.
- Whether the Service type is `NodePort`.

### Step 3: Verify if the Ingress has been properly set up and matched
Confirm:

- Whether the `ingressClassName` is correct.
- Whether the host is correct.
- Whether the path is correct.
- Whether the Host in the request header matches the rules.

### Step 4: Check if the Service is configured correctly
Confirm:

- Whether the backend Service name is correct.
- Whether the backend Service port is correct.

### Step 5: Verify if the Endpoints are valid
Confirm:

- Whether the Service has selected the correct Pod.
- Whether the Pod is Ready.
- Whether the Endpoints are empty.

### Step 6: Verify if the application is functioning normally
Confirm:

- Whether the Pod is running correctly.
- Whether the container process is listening on the correct port.
- Whether the application itself can return a normal response.

---

## IV. First Category of Issues: Complete Unavailability to Access

### Common Symptoms

For example:

- The browser cannot open the page.
- curl encounters a timeout.
- The connection is refused.
- No HTTP response is returned.

### Troubleshooting Order

#### 1. First, confirm if the Controller is functioning normally
    kubectl get pods -n ingress-nginx
    kubectl get svc -n ingress-nginx

Focus on confirming:

- That `ingress-nginx-controller-xxxxx` is `Running`.
- That `ingThe 🔤 symbols have been removed from the text. Here is the translated content in English:

```
kubectl run test-shell --rm -it --image=busybox:1.36 -n <namespace> -- sh

After entering, execute the following command:

wget -qO- http://<service-name>:<port>

If accessing the Service from within the cluster fails, then the issue lies not with the Ingress but with the back-end service chain.
---

## Section 9: The Fourth Type of Problem: The Ingress Has Been Created, But No Traffic Hits the Back End

### Common Symptoms

For example:

- The Ingress exists.
- The Pod and Service are functioning normally.
- However, there are no requests recorded in the business logs.
- It seems as if the back end is not receiving any traffic.

This usually indicates that:

> The request may not have matched any of the current Ingress rules at all.

### Common Causes

- The request did not reach the Controller.
- The Host does not match.
- The Path does not match.
- Other Ingress rules are interfering.
---

## Section 10: Methods for Troubleshooting When There Is No Traffic on the Back End

### 1. Check the Controller Logs

    kubectl logs -n ingress-nginx <controller-pod-name>

If there are no corresponding request records in the logs, first check:

- hosts
- Node IP addresses
- NodePort numbers
- The network connectivity between the local machine and the node.

### 2. Verify If the Host Is Correct

In the current environment, it is recommended to always use:

    curl -H "Host: nginx.example.com" http://10.0.0.20:31152

This will ensure that the request targets the correct Host rules.

### 3. Check If the Path Matches

If the Ingress only handles:

    /api

but the actual access attempt is for:

    /

then the rule will not be triggered.

### 4. Verify If There Are Other Ingress Rules Affecting It

    kubectl get ingress -A

When there are multiple Ingresses in the cluster, it is important to review all the rules together, rather than just focusing on a single YAML file.
---

## Section 11: A Practical Habit: Forced Layer-by-Layer Verification

When the problem symptoms are complex, it is helpful to perform forced layer-by-layer verification.

### Step 1: Verify the Application Itself
Ensure that the Pod, container, and processes are functioning normally.

### Step 2: Verify the Service
Access it directly from within the cluster:

    http://service-name:port

### Step 3: Verify the Controller Entrance
Test directly:

    curl http://10.0.0.20:31152
    curl http://10.0.0.21:31152
    curl http://10.0.0.22:31152

### Step 4: Verify the Ingress Rules
Test with a Host header:

    curl -H "Host: nginx.example.com" http://10.0.0.20:31152

### Step 5: Verify the Hosts or Domain Names
Finally, verify:

    curl http://nginx.example.com:31152

This approach helps to quickly narrow down the possible causes of the problem.
---

## Section 12: Why `kubectl get all` Does Not Show Ingresses

A common scenario at this stage is:

    kubectl get all -n test

In this case, you will not see:

- Ingresses

This is normal.  
The reason is that:

> By default, `kubectl get all` does not display Ingresses.

If you want to view them together, you should execute:

    kubectl get ingress -n test

or:

    kubectl get pod, svc,deploy,rs,ing -n test
---

## Section 13: The Most Commonly Used Set of Commands for Troubleshooting

### Check the Controller

    kubectl get pods -n ingress-nginx
    kubectl get svc -n ingress-nginx
    kubectl logs -n ingress-nginx <controller-pod-name>

### Check the Ingress

    kubectl get ingress -A
    kubectl get ingress -n <namespace> -o wide
    kubectl describe ingress <ingress-name> -n <namespace>
    kubectl get ingress -n <namespace> -o yaml

### Check the IngressClass

    kubectl get ingressclass

### Check the Back-End Service and Endpoints

    kubectl get svc -n <namespace>
    kubectl get endpoints -n <namespace>
    kubectl describe svc <service-name> -n <namespace>

### Check the Back-End Pod