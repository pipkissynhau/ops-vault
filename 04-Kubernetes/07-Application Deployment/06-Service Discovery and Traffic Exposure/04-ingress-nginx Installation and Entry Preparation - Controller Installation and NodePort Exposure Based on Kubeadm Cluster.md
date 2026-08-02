# 04-ingress-nginx Installation and Entry Preparation: Controller Installation and NodePort Exposure Based on Kubeadm Cluster

## Document Description
- Documentation Location: This section focuses on the installation preparation for Ingress during the Kubernetes service discovery and traffic exposure process.
- Applicable Phase: After understanding Services, NodePorts, the need for Ingress, and why having only Ingress YAML is not sufficient, you can begin installing the ingress-nginx Controller in your Kubeadm cluster.
- Recommended Path: `04-Kubernetes/07-Application Deployment/06-Service Discovery and Traffic Exposure/04-ingress-nginx Installation and Entry Preparation: Controller Installation and NodePort Exposure Based on Kubeadm Cluster.md`

## Tags
#Kubernetes #Ingress #IngressController #ingress-nginx #kubeadm #NodePort #IngressClass #Application Deployment #Cloud-Native #Ops #Interview Notes

---

## I. Objectives of This Document

This document focuses on the installation and entry preparation of ingress-nginx in a self-built Kubeadm cluster. It only includes the necessary steps for installation and provides explanations for the key resources after installation.

After completing this document, you should be able to:

- Download the official bare-metal deployment checklist for ingress-nginx.
- Complete the installation of the ingress-nginx Controller.
- Verify that the Controller Pod, Service, and IngressClass are functioning properly.
- Identify the NodePorts corresponding to HTTP and HTTPS.
- Understand the role of the key resources after installation.

---

## II. Prerequisites

The default environment for this document is as follows:

- The Kubernetes cluster is deployed using Kubeadm.
- The current environment does not rely on cloud provider `LoadBalancer`.
- The Ingress Controller used is `ingress-nginx`.
- At this stage, entry exposure is understood in terms of `NodePort`.

---

## III. Installation Steps

### 1. Create a Working Directory

    mkdir -p ~/k8s/ingress-nginx
    cd ~/k8s/ingress-nginx

### 2. Download the Official Bare-Metal Deployment Checklist

    wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/baremetal/deploy.yaml

After downloading, the `deploy.yaml` file should be in the current directory.

### 3. Perform Installation

    kubectl apply -f deploy.yaml

---

## IV. Post-Installation Verification

### 1. Check Pods

    kubectl get pods -n ingress-nginx

Key points to confirm at this stage:

- The `ingress-nginx-controller-xxxxx` Pod exists.
- Its status is `Running`.
- It is `READY`.

### 2. Check the Service

    kubectl get svc -n ingress-nginx

Key points to confirm:

- The `ingress-nginx-controller` Service exists.
- Its type is `NodePort`.
- It has been assigned the corresponding HTTP and HTTPS ports.

The actual results in the current environment may look like this:

    NAME                                         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)
    ingress-nginx-controller                     NodePort    10.106.166.208   <none>        80:31152/TCP,443:30361/TCP
    ingress-nginx-controller-admission           ClusterIP   10.102.34.234    <none>        443/TCP

Record the following:

- HTTP NodePort: `31152`
- HTTPS NodePort: `30361`

### 3. Check the IngressClass

    kubectl get ingressclass

The actual results in the current environment may look like this:

    NAME    CONTROLLER             PARAMETERS   AGE
    nginx   k8s.io/ingress-nginx   <none>       14m

Key points to confirm:

- The `nginx` IngressClass exists.
- The `CONTROLLER` is `k8s.io/ingress-nginx`.

This means that subsequent business Ingresses can typically be defined as follows:

    ingressClassName: nginx

---

## V. Current Entry Addresses in the Environment

The current cluster nodes are:

- `k8s-master`: `10.0.0.20`
- `k8s-node01`: `10.0.0.21`
- `k8s-node02`: `10.0.0.22`

Since the `ingress-nginx-controller` Service uses `NodePort`, the port addresses are:

- HTTP: `31152`
- HTTPS: `30361`

Therefore, the current entry addresses are:

### HTTP Entry
    http://10.0.0### Key Points to Clarify at This Stage

- This Service is not intended for use by browsers, curl, or business domain names.
- This Service will not serve as an external entry point for subsequent business Ingresses.
- It is normal to observe its existence.

---

### 5. Deployment

For example:

    deployment.apps/ingress-nginx-controller

Function:

- Manages the ingress-nginx Controller Pod.
- Ensures the Pod remains active continuously.
- Automatically restarts a new Pod in case of anomalies.
- Supports subsequent rolling updates.

A Deployment is an object that manages the workload of the entry controller, while a Pod is the actual running instance created by it.

---

### 6. ReplicaSet

For example:

    replicaset.apps/ingress-nginx-controller-xxxx

Function:

- Maintains the required number of Pod replicas according to the Deployment's specifications.

At this stage, it can be understood as a layer of replica management beneath the Deployment.

---

## VII. Resources Most Needing Attention at This Stage

After installation, the following resources should be given special attention:

### 1. IngressClass
Determine which Controller the subsequent business Ingress should be assigned to.

### 2. Controller Pod
Verify whether the entry controller is actually running.

### 3. Controller Service
Identify where external requests will enter.

### 4. HTTP / HTTPS NodePort
Confirm the port numbers used for access.

### 5. admission Service
Understand that it is part of the validation process and not an actual business entry point.

---

## VIII. Phase Conclusion

After completing the installation and checking the above resources, the following conclusions can be drawn:

- ingress-nginx has been successfully installed.
- `IngressClass/nginx` exists.
- `ingress-nginx-controller` is the current entry Service in this environment.
- The current entry method is through `NodePort`.
- The HTTP entry port is `31152`, and the HTTPS entry port is `30361`.
- `ingress-nginx-controller-admission` is an internal Service related to Ingress validation.
- Subsequent business Ingresses should use:

    ingressClassName: nginx

    to be managed by the ingress-nginx Controller.

---

## Quick Reference Terms

- ingress-nginx: A common implementation of an Ingress Controller.
- Controller Pod: The program that executes entry rules.
- Controller Service: The entrance for external requests to reach the Controller.
- IngressClass: Specifies which Controller the Ingress rules should be assigned to.
- NodePort: The current method used for exposing the entry points.
- ClusterIP: An internal address within the cluster, not the main external entry point at this stage.

---

## Next Steps Suggested for Tomorrow

The next article suggests organizing the following content:

[[05-First Practical Use of Ingress: Exposing Nginx Services Using ingress-nginx]]