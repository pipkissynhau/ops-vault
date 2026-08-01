# 04-ingress-nginx Installation and Ingress Preparation: Installing the Controller and Exposing NodePort for kubeadm Cluster

## Document Notes
- Document Positioning: Kubernetes Service Discovery and Traffic Exposure Phase, Ingress Learning Installation Preparation Section
- Applicable Stage: After understanding Service, NodePort, why Ingress is needed, and recognizing that only Ingress YAML is insufficient, begin installing ingress-nginx Controller in kubeadm cluster
- Recommended Path: `04-Kubernetes/07-Apply deployment/06-Service detection and traffic exposure/04-ingress-nginx Installation and port readiness: based on kubeadm Cluster Controller Install and NodePort Exposure.md`

## Tags
#Kubernetes #Ingress #IngressController #ingress-nginx #kubeadm #NodePort #IngressClass #ApplyDeployment #Clouds. #Transport #InterviewNotes

---

## I. Article Objectives

This article focuses on ingress-nginx installation and ingress preparation in kubeadm self-hosted clusters, retaining only necessary installation steps and supplementing explanations of key post-installation resources.

After completing this article, the following objectives should be achieved:

- Ability to download ingress-nginx official bare-metal deployment manifest
- Ability to complete ingress-nginx Controller installation
- Ability to check Controller Pod, Service, IngressClass status
- Ability to confirm HTTP and HTTPS corresponding NodePort
- Ability to understand key post-installation resources and their responsibilities

---

## II. Environmental Prerequisites

This article assumes the following environment:

- Kubernetes cluster deployed via kubeadm
- Current environment does not depend on cloud vendors `LoadBalancer`
- Ingress Controller uses `ingress-nginx`
- Current stage ingress exposure method is understood as `NodePort`

---

## III. Installation Steps

### 1. Create Working Directory

    mkdir -p ~/k8s/ingress-nginx
    cd ~/k8s/ingress-nginx

### 2. Download Official bare-metal Deployment Manifest

    wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/baremetal/deploy.yaml

After download, the current directory should contain:

    deploy.yaml

### 3. Execute Installation

    kubectl apply -f deploy.yaml

---

## IV. Post-Installation Checks

### 1. Check Pod

    kubectl get pods -n ingress-nginx

Current stage focus verification:

- `ingress-nginx-controller-xxxxx` exists
- Status is `Running`
- `READY` is normal

### 2. Check Service

    kubectl get svc -n ingress-nginx

Focus verification:

- `ingress-nginx-controller` exists
- Type is `NodePort`
- HTTP and HTTPS corresponding ports are assigned

Current environment actual results:

    NAME                                         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)
    ingress-nginx-controller                     NodePort    10.106.166.208   <none>        80:31152/TCP,443:30361/TCP
    ingress-nginx-controller-admission           ClusterIP   10.102.34.234    <none>        443/TCP

Should record:

- HTTP NodePort: `31152`
- HTTPS NodePort: `30361`

### 3. Check IngressClass

    kubectl get ingressclass

Current environment actual results:

    NAME    CONTROLLER             PARAMETERS   AGE
    nginx   k8s.io/ingress-nginx   <none>       14m

Focus verification:

- `nginx` exists
- `CONTROLLER` is `k8s.io/ingress-nginx`

This means subsequent business Ingress can typically be written as:

    ingressClassName: nginx

---

## V. Current Environment Ingress Addresses

Current cluster nodes:

- `k8s-master`: `10.0.0.20`
- `k8s-node01`: `10.0.0.21`
- `k8s-node02`: `10.0.0.22`

Current `ingress-nginx-controller` Service type is `NodePort`, ports are:

- HTTP: `31152`
- HTTPS: `30361`

Therefore, current stage ingress addresses can be understood as:

### HTTP Ingress
    http://10.0.0.20:31152
    http://10.0.0.21:31152
    http://10.0.0.22:31152

### HTTPS Ingress
    https://10.0.0.20:30361
    https://10.0.0.21:30361
    https://10.0.0.22:30361

---

## VI. Key Post-Installation Resources Explanation

After installing ingress-nginx, common resources like `IngressClass`, Pod, Service, Deployment, ReplicaSet may be seen.  
This stage does not require expanding all controller principles, but needs to clarify the responsibilities of these resources.

### 1. IngressClass

Example:

    nginx   k8s.io/ingress-nginx   <none>

Function:

- Indicates the cluster has an IngressClass named `nginx`
- The IngressClass corresponding Controller is:

    k8s.io/ingress-nginx

Subsequent business Ingress typically belongs to this Controller via:

    ingressClassName: nginx

Additional Notes:

- `PARAMETERS` showing as `<none>` indicates no additional parameters object is associated with this IngressClass
- This is a normal phenomenon and does not affect subsequent usage

This is the actual runtime entity that executes traffic forwarding for the entry controller.

---

### 3. Service: ingress-nginx-controller

Example:

    service/ingress-nginx-controller   NodePort   10.106.166.208   <none>   80:31152/TCP,443:30361/TCP

Type:

    NodePort

Purpose:

- Provides a stable entry point for ingress-nginx Controller
- Introduces external requests to Controller Pod
- Exposes NodePort for HTTP and HTTPS

Current environment focus:

- HTTP NodePort: `31152`
- HTTPS NodePort: `30361`

Notes:

- `10.106.166.208` is the ClusterIP of this Service, only for internal cluster access
- The external entry point in this phase is not ClusterIP, but:

    NodeIP:31152
    NodeIP:30361

Therefore, the foundation for subsequent business Ingress entries should be understood as:

### HTTP Entry
    http://10.0.0.20:31152
    http://10.0.0.21:31152
    http://10.0.0.22:31152

### HTTPS Entry
    https://10.0.0.20:30361
    https://10.0.0.21:30361
    https://10.0.0.22:30361

---

### 4. Service: ingress-nginx-controller-admission

Example:

    service/ingress-nginx-controller-admission   ClusterIP   10.102.34.234   <none>   443/TCP

Type:

    ClusterIP

Purpose:

- Provides cluster internal access entry for ingress-nginx's admission webhook
- Allows Kubernetes to call webhook for validation when creating or updating Ingress resources
- Serves the control plane validation process, not involved in business traffic forwarding

Understanding approach:

- `ingress-nginx-controller` is the business traffic entry
- `ingress-nginx-controller-admission` is the Ingress validation entry

Current phase clarification:

- This Service is not for browser, curl, or business domain access
- This Service is not the external entry for subsequent business Ingress
- Seeing its existence is a normal phenomenon

---

### 5. Deployment

Example:

    deployment.apps/ingress-nginx-controller

Purpose:

- Manages ingress-nginx Controller Pod
- Ensures Pod remains continuously available
- Automatically restarts new Pod when Pod fails
- Supports subsequent rolling updates

Deployment is the management object for the entry controller's workloads, and Pod is its actual created runtime instance.

---

### 6. ReplicaSet

Example:

    replicaset.apps/ingress-nginx-controller-xxxx

Purpose:

- Maintains the number of current version Pod replicas according to Deployment requirements

Currently, it can be understood as a layer of replica management object under Deployment.

---

## Seven. Resources Most Need to Focus On in Current Phase

After installation, the following resource types are most critical to focus on in the current phase:

### 1. IngressClass
Confirm which Controller the subsequent business Ingress should belong to

### 2. Controller Pod
Confirm whether the entry controller is actually running

### 3. Controller Service
Confirm where external requests enter

### 4. HTTP / HTTPS NodePort
Confirm the entry port for subsequent access

### 5. admission Service
Clarify it belongs to the validation process, not business entry

---

## Eight. Phase Conclusion

After completing installation and checking the above resources, the current phase should form the following conclusions:

- ingress-nginx has been installed
- `IngressClass/nginx` exists
- `ingress-nginx-controller` is the entry Service of the current environment
- The current entry exposure method is `NodePort`
- HTTP entry port is `31152`
- HTTPS entry port is `30361`
- `ingress-nginx-controller-admission` belongs to the internal Service related to Ingress validation
- Subsequent business Ingress should be submitted via:

    ingressClassName: nginx

to be taken over by ingress-nginx Controller

---

## Keyword Quick Notes

- ingress-nginx: Common Ingress Controller implementation
- Controller Pod: Program carrier that executes entry rules
- Controller Service: Entry point for external requests to Controller
- IngressClass: Which Controller the Ingress rules belong to
- NodePort: Entry exposure method in current phase
- ClusterIP: Cluster internal access address, not the main external entry in current phase

---

## Next Day Suggestions

Next article suggestion to organize:

[[05-First Ingress Hands-on - Exposing Nginx Service with ingress-nginx]]