# 02-ingress-nginx Production Installation: NodePort, IngressClass, and Access Verification

Recommended path:

    04-Kubernetes/08-Operations/02-Cluster Base Component Installation/02-ingress-nginx Production Installation: NodePort, IngressClass, and Access Verification.md

Tags:

    #Kubernetes
    #Ingress
    #ingress-nginx
    #IngressClass
    #NodePort
    #7thFloorEntrance.
    #ServiceExposure
    #ClusterBasicComponents

---

## I. Document Description

This document records the installation, verification, and common troubleshooting methods for ingress-nginx in a Kubernetes cluster.

ingress-nginx is a common seven-layer ingress controller in Kubernetes, used to forward external HTTP/HTTPS requests to internal cluster Services based on Ingress rules.

Deployment method in this document:

    Installation method: Helm
    Controller: ingress-nginx
    Exposure method: NodePort
    HTTP NodePort: 30080
    HTTPS NodePort: 30443
    IngressClass: nginx
    Replica count: 2
    Applicable environment: kubeadm self-built cluster, bare-metal cluster, private environment

Execution node:

    k8s-master-01

Prerequisites:

    1. Kubernetes cluster has been deployed
    2. Node status is Ready
    3. CNI is normal
    4. CoreDNS is normal
    5. kube-proxy is normal
    6. Helm is installed
    7. kubectl can access the cluster normally

---

## II. Relationship Between Ingress and ingress-nginx

Ingress is a Kubernetes resource object that only defines access rules.

ingress-nginx is an Ingress Controller that is responsible for actually listening to external traffic and forwarding it to backend Services.

Relationship:

    User request
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
    Ingress rules
       |
       v
    Service
       |
       v
    Pod

Note:

    Only Ingress YAML is not enough.
    A Ingress Controller must exist for Ingress rules to take effect.

---

## III. Entry Address Planning

This document uses NodePort to expose ingress-nginx.

| Item | Planning |
|---|---|
| ingress-nginx namespace | ingress-nginx |
| IngressClass | nginx |
| HTTP NodePort | 30080 |
| HTTPS NodePort | 30443 |
| Test domain | demo.ops.local |
| Test service | nginx-demo |
| Test namespace | demo |

Access example:

    http://AnyNodeIP:30080
    https://AnyNodeIP:30443

Example:

    http://10.0.0.23:30080
    http://10.0.0.24:30080
    http://10.0.0.25:30080

Note:

    10.0.0.30 is the kube-vip address of the Kubernetes APIServer.
    It is not recommended to use the APIServer VIP directly as a business Ingress entry.
    In production environments, it is recommended to plan separate load balancers, SLB, F5, Nginx, HAProxy, MetalLB, or independent business VIPs for business entries.

---

## IV. Pre-deployment Checks

### 4.1 Check Cluster Nodes

Execute:

    kubectl get nodes -o wide

Requirements:

    Master Ready
    Worker Ready

---

### 4.2 Check Helm

Execute:

    helm version

If Helm is not present, install Helm first.

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

Default Kubernetes NodePort range:

    30000-32767

This document uses:

    30080
    30443

Check if the port is occupied:

    sudo ss -lntp | grep -E "30080|30443"

If there is no output, it means no process is directly occupying the port on the host.

---

## V. Add ingress-nginx Helm Repository

Add the official Helm repository:

    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

Update the repository:

    helm repo update

View the repository:

    helm repo list

Search for the Chart:

    helm search repo ingress-nginx

If the current environment has slow access to GitHub Pages, you can pre-execute on a machine with network access:

    helm pull ingress-nginx/ingress-nginx

Then upload the chart package to the cluster management node and install it offline.

---

## VI. Prepare values File

Create directory:

    mkdir -p /root/k8s-yaml/ingress-nginx

    cd /root/k8s-yaml/ingress-nginx

Create values file:

    cat <<EOF > values-ingress-nginx-nodeport.yaml
    controller:
      replicaCount: 2

      ingressClass: nginx /think

```yaml
ingressClassResource:
  name: nginx
  enabled: true
  default: false
  controllerValue: k8s.io/ingress-nginx

watchIngressWithoutClass: false

service:
  type: NodePort
  externalTrafficPolicy: Cluster
  nodePorts:
    http: 30080
    https: 30443

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

admissionWebhooks:
  enabled: true
```

**Explanation:**

- replicaCount: 2  
  Start two ingress-nginx-controller replicas to improve the availability of the ingress component.

- ingressClassResource.name: nginx  
  Create an IngressClass named nginx.

- watchIngressWithoutClass: false  
  Only process Ingress resources that explicitly specify ingressClassName: nginx.

- service.type: NodePort  
  Expose HTTP/HTTPS ingress using NodePort.

- externalTrafficPolicy: Cluster  
  Traffic can access any NodePort node, then forwarded to ingress-nginx-controller Pod by kube-proxy.

**Production Notes:**

- externalTrafficPolicy: Cluster  
  **Advantages:** Any NodePort node is accessible.  
  **Disadvantages:** May not preserve the client's real source IP.

- externalTrafficPolicy: Local  
  **Advantages:** Preserves the client's real source IP.  
  **Disadvantages:** External load balancer must only forward traffic to nodes with ingress-nginx-controller Pod, otherwise access may fail.

- This document first uses Cluster to ensure a more stable deployment and verification process.

---

## VII. Installing ingress-nginx

Create a namespace:

```bash
kubectl create namespace ingress-nginx
```

Install using Helm:

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  -n ingress-nginx \
  -f values-ingress-nginx-nodeport.yaml
```

Check Helm release:

```bash
helm list -n ingress-nginx
```

Check resources:

```bash
kubectl -n ingress-nginx get all -o wide
```

Check IngressClass:

```bash
kubectl get ingressclass
```

Expected output:

```
nginx
```

---

## VIII. Checking ingress-nginx Status

### 8.1 View Pods

Run:

```bash
kubectl -n ingress-nginx get pods -o wide
```

Expected:

```
ingress-nginx-controller Pod Running
```

If replica count is 2, two controller Pods should be visible.

---

### 8.2 View Service

Run:

```bash
kubectl -n ingress-nginx get svc
```

Expected output:

```
ingress-nginx-controller
```

Ports similar to:

```
80:30080/TCP
443:30443/TCP
```

Check detailed information:

```bash
kubectl -n ingress-nginx describe svc ingress-nginx-controller
```

---

### 8.3 View Controller Logs

Run:

```bash
kubectl -n ingress-nginx logs deploy/ingress-nginx-controller
```

If multiple replicas exist, you can also view all controller logs:

```bash
kubectl -n ingress-nginx logs -l app.kubernetes.io/component=controller --tail=100
```

---

### 8.4 View Admission Job

The ingress-nginx installation typically creates admission webhook-related Jobs.

Check:

```bash
kubectl -n ingress-nginx get jobs
kubectl -n ingress-nginx get pods | grep admission
```

If the Job fails, it's usually related to image pull, network access, or webhook certificate generation.

---

## IX. Deploying Test Application

### 9.1 Create Test Namespace

Run:

```bash
kubectl create namespace demo
```

---

### 9.2 Create nginx Test Application

Run:

```bash
kubectl -n demo create deployment nginx-demo --image=nginx:1.25
```

Scale to 2 replicas:

```bash
kubectl -n demo scale deployment nginx-demo --replicas=2
```

Check Pods:

```bash
kubectl -n demo get pods -o wide
```

---

### 9.3 Create Service

Run:

```bash
kubectl -n demo expose deployment nginx-demo \
  --port=80 \
  --target-port=80 \
  --type=ClusterIP
```

Check Service:

```bash
kubectl -n demo get svc
```

Check Endpoints:

```bash
kubectl -n demo get endpoints nginx-demo
```

Endpoints must not be empty.

If Endpoints is empty, it indicates that the Service selector did not match any Pod.

---

## 10. Creating Ingress Rules

Create Ingress:

    cat <<EOF > ingress-nginx-demo.yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: nginx-demo
      namespace: demo
    spec:
      ingressClassName: nginx
      rules:
      - host: demo.ops.local
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-demo
                port:
                  number: 80
    EOF

Apply:

    kubectl apply -f ingress-nginx-demo.yaml

Check Ingress:

    kubectl -n demo get ingress

Check detailed information:

    kubectl -n demo describe ingress nginx-demo

Confirm:

    IngressClass is nginx
    Backend Service is nginx-demo:80
    No obvious error events

---

## 11. Access Verification

### 11.1 Access Using curl with Specified Host

On any machine that can access NodeIP, execute:

    curl -H "Host: demo.ops.local" http://10.0.0.23:30080/

You can also test other nodes:

    curl -H "Host: demo.ops.local" http://10.0.0.24:30080/

    curl -H "Host: demo.ops.local" http://10.0.0.25:30080/

If it returns the nginx default page, the access is successful.

---

### 11.2 Access After Configuring Local hosts

Configure hosts on the access client:

    10.0.0.23 demo.ops.local

Then access:

    curl http://demo.ops.local:30080/

Browser access:

    http://demo.ops.local:30080/

Note:

    If using NodePort, the port must include 30080.
    If there is an external entry point like Nginx, SLB, F5, or HAProxy in production, it can listen on 80/443 and forward to NodePort 30080/30443.

---

### 11.3 View ingress-nginx Logs

Execute:

    kubectl -n ingress-nginx logs -l app.kubernetes.io/component=controller --tail=100

If access is successful, you can typically see request records in the logs.

---

## 12. HTTPS Test Notes

This document primarily verifies HTTP first.

HTTPS access depends on TLS Secret, for example:

    tls.crt
    tls.key

Manual creation of TLS Secret example:

    kubectl -n demo create secret tls demo-tls \
      --cert=tls.crt \
      --key=tls.key

Reference in Ingress:

    spec:
      tls:
      - hosts:
        - demo.ops.local
        secretName: demo-tls

Production environment recommendations:

    Use cert-manager to manage TLS certificates.
    cert-manager can automatically apply, issue, and renew certificates, and maintain Kubernetes Secrets automatically.
    cert-manager is recommended to be documented separately.

---

## 13. Production Ingress Planning Recommendations

### 13.1 NodePort Method

Suitable for:

    1. kubeadm self-built clusters
    2. Bare-metal clusters
    3. Private environments
    4. Experimental environments
    5. Production environments with external load balancers

Typical flow:

    User
      |
      v
    External LB / Nginx / F5 / SLB
      |
      v
    NodeIP:30080 / NodeIP:30443
      |
      v
    ingress-nginx-controller
      |
      v
    Service
      |
      v
    Pod

---

### 13.2 LoadBalancer Method

In cloud vendor-managed Kubernetes, Service type can directly use:

    LoadBalancer

In bare-metal environments, if you want to use LoadBalancer type, you generally need to install:

    MetalLB

This section is recommended to be organized separately and not included in the main document.

---

### 13.3 Not Recommended to Mix APIServer VIP

In this series:

    10.0.0.30 = k8s-api-server = APIServer VIP

This VIP is used for Kubernetes control plane access and is not recommended as a business entry VIP.

In production, it is recommended to plan separately:

    Control plane VIP
    Business entry VIP
    Internal service entry
    External service entry

---

## 14. Common Troubleshooting

### 14.1 ingress-nginx-controller Pod ImagePullBackOff

Check Pod:

    kubectl -n ingress-nginx get pods -o wide

Check events:

    kubectl -n ingress-nginx describe pod <pod-name>

Common causes:

    1. Unable to access registry.k8s.io
    2. Unable to access GitHub-related image repositories
    3. Images not synchronized to internal Harbor
    4. containerd unable to pull images

Troubleshooting steps:

    1. Confirm node network
    2. Check image address
    3. Synchronize images to internal Harbor
    4. Modify image configuration in Helm values
    5. Reapply helm upgrade /think

View current Chart default values:

    helm show values ingress-nginx/ingress-nginx > values-default.yaml

Check image-related configurations:

    grep -n "registry:" values-default.yaml

    grep -n "image:" values-default.yaml

    grep -n "tag:" values-default.yaml

---

### 14.2 Ingress Access Returns 404

Phenomenon:

    curl returns 404 Not Found

Troubleshooting:

    kubectl -n demo get ingress

    kubectl -n demo describe ingress nginx-demo

    kubectl get ingressclass

Checkpoints:

    1. Does the Host match?
    2. Does the path match?
    3. Is the ingressClassName set to nginx?
    4. Does the request include the correct Host?
    5. Is the Ingress recognized by ingress-nginx?

Common errors:

    curl http://10.0.0.23:30080/

This request lacks Host: demo.ops.local, which may prevent matching the Ingress rule.

Correct way:

    curl -H "Host: demo.ops.local" http://10.0.0.23:30080/

---

### 14.3 Ingress Access Returns 502 / 503

Troubleshoot Service:

    kubectl -n demo get svc nginx-demo

Troubleshoot Endpoints:

    kubectl -n demo get endpoints nginx-demo

Troubleshoot Pod:

    kubectl -n demo get pods -o wide

    kubectl -n demo describe pod <pod-name>

Common causes:

    1. Incorrect Service targetPort
    2. Service selector does not match Pod label
    3. Pod is not Running
    4. Application in Pod is not listening on the corresponding port
    5. Empty Endpoints

---

### 14.4 NodePort Access Fails

Check Service:

    kubectl -n ingress-nginx get svc ingress-nginx-controller

Check NodePort:

    kubectl -n ingress-nginx describe svc ingress-nginx-controller

Check node ports:

    sudo ipvsadm -Ln | grep -E "30080|30443"

Check kube-proxy:

    kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide

Check firewall:

    sudo ufw status

Check port connectivity:

    curl -I http://10.0.0.23:30080/

Common causes:

    1. Incorrect NodePort
    2. Security group or firewall not allowing traffic
    3. kube-proxy anomaly
    4. ingress-nginx-controller Pod not running
    5. externalTrafficPolicy configuration mismatch with access node

---

### 14.5 IngressClass Mismatch

Check IngressClass:

    kubectl get ingressclass

Check Ingress:

    kubectl -n demo get ingress nginx-demo -o yaml | grep ingressClassName

Requirement:

    ingressClassName: nginx

If no ingressClassName is specified and the controller is configured with:

    watchIngressWithoutClass: false

Then the Ingress will not be processed.

Resolution:

    Explicitly add to Ingress:

        ingressClassName: nginx

---

### 14.6 Admission Webhook Error

Common phenomena:

    failed calling webhook
    validating webhook
    certificate signed by unknown authority

Troubleshooting:

    kubectl -n ingress-nginx get jobs

    kubectl -n ingress-nginx get pods | grep admission

    kubectl -n ingress-nginx get validatingwebhookconfiguration

    kubectl -n ingress-nginx describe pod <admission-pod-name>

Common causes:

    1. Admission Job failed
    2. webhook certificate not generated
    3. image pull failure
    4. installation interruption caused webhook residue

Handling approach:

    1. Check admission Job logs
    2. Re-run helm upgrade
    3. Uninstall and reinstall if necessary
    4. Check for residual validatingwebhookconfiguration

---

## Fifteen. Upgrade and Rollback

### 15.1 Check Current Version

Check Helm release:

    helm list -n ingress-nginx

Check Chart information:

    helm status ingress-nginx -n ingress-nginx

Check controller image:

    kubectl -n ingress-nginx get deploy ingress-nginx-controller -o yaml | grep image:

---

### 15.2 Upgrade ingress-nginx

After modifying values, execute:

    helm upgrade ingress-nginx ingress-nginx/ingress-nginx \
      -n ingress-nginx \
      -f values-ingress-nginx-nodeport.yaml

Check upgrade status:

helm history ingress-nginx -n ingress-nginx

---

### 15.3 Rollback ingress-nginx

Check history versions:

    helm history ingress-nginx -n ingress-nginx

Rollback to a specific version:

    helm rollback ingress-nginx <REVISION> -n ingress-nginx

Check status:

    helm status ingress-nginx -n ingress-nginx

---

## SixteenI don't know.Uninstall ingress-nginx

Uninstall Helm release:

    helm uninstall ingress-nginx -n ingress-nginx

Delete namespace:

    kubectl delete namespace ingress-nginx

Check residual IngressClass:

    kubectl get ingressclass

If deletion of IngressClass is needed:

    kubectl delete ingressclass nginx

Check residual webhook:

    kubectl get validatingwebhookconfiguration | grep ingress

If residual webhook is confirmed and no longer in use, it can be manually deleted.

---

## SeventeenI don't know.Post-Installation Checklist

After installation, execute:

    helm list -n ingress-nginx

    kubectl -n ingress-nginx get pods -o wide

    kubectl -n ingress-nginx get svc

    kubectl get ingressclass

    kubectl -n demo get ingress

    kubectl -n demo get svc

    kubectl -n demo get endpoints

    curl -H "Host: demo.ops.local" http://10.0.0.23:30080/

Should satisfy:

    1. ingress-nginx-controller Pod is Running
    2. ingress-nginx-controller Service type is NodePort
    3. HTTP NodePort is 30080
    4. HTTPS NodePort is 30443
    5. IngressClass nginx exists
    6. Test Ingress's ingressClassName is nginx
    7. Backend Service Endpoints is not empty
    8. curl with specified Host can access backend nginx page

---

## EighteenI don't know.Summary

This document completes the basic production installation of ingress-nginx.

Core content:

    1. Install ingress-nginx using Helm
    2. Expose HTTP / HTTPS using NodePort
    3. Fix HTTP NodePort to 30080
    4. Fix HTTPS NodePort to 30443
    5. Create IngressClass nginx
    6. Create test application
    7. Create Ingress rules
    8. Verify access using Host header
    9. Troubleshoot common issues like 404I don't know.502I don't know.503I don't know.NodePort not accessibleI don't know.IngressClass mismatch, etc.

Recommended next steps to continue organizing:

    03-cert-manager installation: TLS certificate automatic issuanceI don't know.renewal and Secret management.md /think