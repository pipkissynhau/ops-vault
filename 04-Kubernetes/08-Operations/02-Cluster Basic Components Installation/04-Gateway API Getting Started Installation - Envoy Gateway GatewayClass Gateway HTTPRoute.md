# 04-Gateway API Getting Started Installation: Envoy Gateway, GatewayClass, Gateway, and HTTPRoute

Recommended path:

    04-Kubernetes/08-Operations/02-Cluster Base Component Installation/04-Gateway API Getting Started Installation: Envoy Gateway, GatewayClass, Gateway, and HTTPRoute.md

Tags:

    #Kubernetes
    #GatewayAPI
    #EnvoyGateway
    #GatewayClass
    #Gateway
    #HTTPRoute
    #7thFloorEntrance.
    #ClusterBasicComponents

---

## I. Document Notes

This document records the basic installation and verification methods for Gateway API in a Kubernetes cluster.

This document uses:

    Gateway API
    Envoy Gateway
    GatewayClass
    Gateway
    HTTPRoute
    NodePort method for access verification

This document's objectives:

    1. Install Envoy Gateway
    2. Create GatewayClass
    3. Create Gateway
    4. Create HTTPRoute
    5. Verify HTTP access via NodePort
    6. Understand the differences between Gateway API and Ingress
    7. Master common troubleshooting methods for Gateway API

Execution node:

    k8s-master-01

Prerequisites:

    1. Kubernetes cluster has been deployed
    2. Node status is Ready
    3. CoreDNS is functioning normally
    4. CNI is functioning normally
    5. Helm is installed
    6. kubectl can access the cluster normally

---

## II. What is Gateway API

Gateway API is a new generation service entry and traffic routing API for Kubernetes.

Compared to traditional Ingress, it has a clearer resource model and more explicit responsibilities.

Common resources:

| Resource | Function |
|---|---|
| GatewayClass | Defines which Gateway Controller is responsible for processing |
| Gateway | Defines entry listening ports, protocols, domains, etc. |
| HTTPRoute | Defines how HTTP requests are forwarded to backend Service |
| ReferenceGrant | Allows cross-namespace resource referencing |
| TLSRoute | TLS traffic routing |
| GRPCRoute | gRPC traffic routing |

This document will first learn the three basic types of resources:

    GatewayClass
    Gateway
    HTTPRoute

---

## III. Differences Between Gateway API and Ingress

### 3.1 Ingress Model

Common Ingress structure:

    Ingress
      |
      v
    Ingress Controller
      |
      v
    Service
      |
      v
    Pod

Characteristics of Ingress:

    1. Simple model
    2. Widely used in enterprises
    3. Mainly for HTTP / HTTPS layer 7 entry
    4. Many advanced capabilities depend on Controller annotations
    5. Different Controller annotations have significant differences

---

### 3.2 Gateway API Model

Common Gateway API structure:

    GatewayClass
      |
      v
    Gateway
      |
      v
    HTTPRoute
      |
      v
    Service
      |
      v
    Pod

Characteristics of Gateway API:

    1. Separation of entry infrastructure and business routing
    2. Gateway is managed by platform or operations team
    3. HTTPRoute can be managed by business team
    4. Clearer resource responsibilities
    5. More suitable for multi-team, multi-tenant, complex traffic governance

---

### 3.3 Simple Understanding

You can understand it this way:

    Ingress:
        A single resource expresses both entry and routing rules.

    Gateway API:
        Gateway handles entry.
        HTTPRoute handles business routing.
        GatewayClass binds to specific Controller.

---

## IV. Deployment Plan

This document uses Envoy Gateway as the Gateway API Controller.

| Project | Plan |
|---|---|
| Gateway Controller | Envoy Gateway |
| Installation method | Helm |
| Envoy Gateway namespace | envoy-gateway-system |
| Gateway API test namespace | gateway-demo |
| GatewayClass name | eg |
| Gateway name | demo-gateway |
| HTTPRoute name | demo-httproute |
| Test domain | demo-gw.ops.local |
| Test service | nginx-gateway-demo |
| Exposure method | NodePort |
| Access port | Automatically generated NodePort |

Notes:

    The environment in this article is a self-built cluster using kubeadm, which does not inherently support LoadBalancer.
    Therefore, this article uses EnvoyProxy to expose Envoy Gateway's data plane Service as NodePort.
    If the production environment has MetalLB, SLB, F5, or cloud provider LoadBalancer, it can be changed to LoadBalancer method.

---

## V. Pre-Deployment Checks

### 5.1 Check Cluster Status

Execute:

    kubectl get nodes -o wide

Requirements:

    All nodes are Ready.

---

### 5.2 Check Helm

Execute:

    helm version

---

### 5.3 Check Base Components

Execute:

    kubectl get pods -A -o wide

Key confirmations:

    kube-system is normal
    calico-system is normal
    CoreDNS is normal
    kube-proxy is normal

---

### 5.4 Check if Gateway API CRD Exists

Execute:

    kubectl get crd | grep gateway.networking.k8s.io

If there is no output, it indicates that the Gateway API CRD is not yet installed on the current cluster.

Installing the Envoy Gateway Helm Chart will install the required CRD.

---

## Six. Installing Envoy Gateway

### 6.1 Create Directory

Execute:

    mkdir -p /root/k8s-yaml/gateway-api

    cd /root/k8s-yaml/gateway-api

---

### 6.2 Set Version Variables

Execute:

    EG_VERSION=v1.7.2

Explanation:

    This example uses Envoy Gateway v1.7.2.
    In production environments, it should be fixed based on company version baseline, Kubernetes version compatibility, and image accessibility.

---

### 6.3 Install Envoy Gateway Using Helm

Execute:

    helm install eg oci://docker.io/envoyproxy/gateway-helm \
      --version ${EG_VERSION} \
      -n envoy-gateway-system \
      --create-namespace

Wait for Envoy Gateway to be ready:

    kubectl wait --timeout=5m \
      -n envoy-gateway-system \
      deployment/envoy-gateway \
      --for=condition=Available

Check Helm Release:

    helm list -n envoy-gateway-system

Check Pods:

    kubectl -n envoy-gateway-system get pods -o wide

Check CRD:

    kubectl get crd | grep gateway

---

### 6.4 Mirror Notes for Domestic Environments

Envoy Gateway's default image comes from a public mirror.

If image pull fails, check Pod events:

    kubectl -n envoy-gateway-system describe pod <pod-name>

Check current Helm values:

    helm show values oci://docker.io/envoyproxy/gateway-helm \
      --version ${EG_VERSION} > values-envoy-gateway-default.yaml

Check image-related configuration:

    grep -n "image:" values-envoy-gateway-default.yaml

    grep -n "repository:" values-envoy-gateway-default.yaml

Production environment recommendations:

    1. Synchronize Envoy Gateway image to internal Harbor
    2. Synchronize Envoy Proxy image to internal Harbor
    3. Use values file to fix image repository and version
    4. Do not recommend directly relying on public image repositories in production

---

## Seven. Create Test Namespace and Test Application

### 7.1 Create Namespace

Execute:

    kubectl create namespace gateway-demo

---

### 7.2 Create nginx Test Application

Create Deployment:

    kubectl -n gateway-demo create deployment nginx-gateway-demo --image=nginx:1.25

Scale:

    kubectl -n gateway-demo scale deployment nginx-gateway-demo --replicas=2

Create Service:

    kubectl -n gateway-demo expose deployment nginx-gateway-demo \
      --port=80 \
      --target-port=80 \
      --type=ClusterIP

Check:

    kubectl -n gateway-demo get pods -o wide

    kubectl -n gateway-demo get svc

    kubectl -n gateway-demo get endpoints nginx-gateway-demo

Requirements:

    Pod Running
    Service exists
    Endpoints not empty

---

## Eight. Create EnvoyProxy NodePort Configuration

In a kubeadm self-built cluster, there is usually no default LoadBalancer capability.

Therefore, this article first creates EnvoyProxy configuration to let Envoy Gateway create data plane Service using NodePort type.

Create EnvoyProxy:

    cat <<EOF > envoyproxy-nodeport.yaml
    apiVersion: gateway.envoyproxy.io/v1alpha1
    kind: EnvoyProxy
    metadata:
      name: nodeport-envoy-proxy
      namespace: gateway-demo
    spec:
      provider:
        type: Kubernetes
        kubernetes:
          envoyService:
            type: NodePort
          envoyDeployment:
            replicas: 2
    EOF

Apply:

    kubectl apply -f envoyproxy-nodeport.yaml

Check:

    kubectl -n gateway-demo get envoyproxy

Explanation:

    envoyService.type: NodePort
        Let Envoy Gateway create data plane Service using NodePort to expose.

    envoyDeployment.replicas: 2
        Set Envoy data plane replica count to 2, improving entry component availability.

---

## Nine. Create GatewayClass

GatewayClass is used to declare which Controller handles the Gateway.

Create GatewayClass: /think

```bash
cat <<EOF > gatewayclass-eg.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
EOF
```

**Apply:**

```bash
kubectl apply -f gatewayclass-eg.yaml
```

**Check:**

```bash
kubectl get gatewayclass
```

**View details:**

```bash
kubectl describe gatewayclass eg
```

**Explanation:**

```
controllerName: gateway.envoyproxy.io/gatewayclass-controller
    Indicates this GatewayClass is handled by Envoy Gateway.
```

---

## Ten. Creating Gateway

Gateway represents specific ingress listener configuration.

This example creates an HTTP listener:

```
Listening port: 80
Protocol: HTTP
Domain: demo-gw.ops.local
```

**Create Gateway:**

```bash
cat <<EOF > gateway-demo.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: demo-gateway
  namespace: gateway-demo
spec:
  gatewayClassName: eg
  infrastructure:
    parametersRef:
      group: gateway.envoyproxy.io
      kind: EnvoyProxy
      name: nodeport-envoy-proxy
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    hostname: demo-gw.ops.local
    allowedRoutes:
      namespaces:
        from: Same
EOF
```

**Apply:**

```bash
kubectl apply -f gateway-demo.yaml
```

**Check:**

```bash
kubectl -n gateway-demo get gateway
```

**View details:**

```bash
kubectl -n gateway-demo describe gateway demo-gateway
```

**Focus on:**

```
Programmed
Accepted
ResolvedRefs
```

**Expected:**

```
Programmed=True
```

**Explanation:**

```
infrastructure.parametersRef
    Indicates this Gateway uses the previously defined EnvoyProxy configuration.

allowedRoutes.namespaces.from: Same
    Indicates only HTTPRoute in the same namespace can bind to this Gateway.
```

---

## Eleven. Creating HTTPRoute

HTTPRoute defines how HTTP requests are forwarded to backend Service.

**Create HTTPRoute:**

```bash
cat <<EOF > httproute-demo.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: demo-httproute
  namespace: gateway-demo
spec:
  parentRefs:
  - name: demo-gateway
  hostnames:
  - demo-gw.ops.local
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: nginx-gateway-demo
      port: 80
EOF
```

**Apply:**

```bash
kubectl apply -f httproute-demo.yaml
```

**Check:**

```bash
kubectl -n gateway-demo get httproute
```

**View details:**

```bash
kubectl -n gateway-demo describe httproute demo-httproute
```

**Focus on:**

```
Accepted
ResolvedRefs
```

**Expected:**

```
Accepted=True
ResolvedRefs=True
```

---

## Twelve. Viewing Envoy Gateway automatically created data plane resources

After creating Gateway, Envoy Gateway automatically creates Envoy data plane Deployment and Service.

**Check Envoy Gateway namespace resources:**

```bash
kubectl -n envoy-gateway-system get deploy,svc,pod -o wide
```

**Find corresponding Envoy Service by Gateway label:**

```bash
kubectl -n envoy-gateway-system get svc \
  -l gateway.envoyproxy.io/owning-gateway-namespace=gateway-demo,gateway.envoyproxy.io/owning-gateway-name=demo-gateway
```

**Save Envoy Service name:**

```bash
export ENVOY_SERVICE=$(kubectl -n envoy-gateway-system get svc \
  -l gateway.envoyproxy.io/owning-gateway-namespace=gateway-demo,gateway.envoyproxy.io/owning-gateway-name=demo-gateway \
  -o jsonpath='{.items[0].metadata.name}')
```

**Check:**

echo ${ENVOY_SERVICE}

Check Service details:

    kubectl -n envoy-gateway-system describe svc ${ENVOY_SERVICE}

Verify:

    Type: NodePort

Get HTTP NodePort:

    export HTTP_NODE_PORT=$(kubectl -n envoy-gateway-system get svc ${ENVOY_SERVICE} \
      -o jsonpath='{.spec.ports[?(@.port==80)].nodePort}')

Check:

    echo ${HTTP_NODE_PORT}

Note:

    NodePort created by Envoy Gateway is automatically assigned by default.
    In production environments, if fixed NodePort is needed, it can be managed separately using methods like EnvoyProxy Service Patch.

---

## Thirteen. Access Verification

### 13.1 Access Using curl with Host Header

Execute on any machine that can access NodeIP:

    curl -H "Host: demo-gw.ops.local" http://10.0.0.23:${HTTP_NODE_PORT}/

You can also test other Worker nodes:

    curl -H "Host: demo-gw.ops.local" http://10.0.0.24:${HTTP_NODE_PORT}/

    curl -H "Host: demo-gw.ops.local" http://10.0.0.25:${HTTP_NODE_PORT}/

If the nginx default page is returned, it indicates successful Gateway API routing.

---

### 13.2 Access After Configuring Local hosts

Configure hosts on the access client:

    10.0.0.23 demo-gw.ops.local

Access:

    curl http://demo-gw.ops.local:${HTTP_NODE_PORT}/

Note:

    This document uses NodePort, so the NodePort port must be included when accessing.
    If an external load balancer is used in front, it can listen on 80/443 and forward to NodePort.

---

### 13.3 View Envoy Gateway Logs

Check control plane logs:

    kubectl -n envoy-gateway-system logs deploy/envoy-gateway --tail=100

Check Envoy data plane Pods:

    kubectl -n envoy-gateway-system get pods \
      -l gateway.envoyproxy.io/owning-gateway-namespace=gateway-demo,gateway.envoyproxy.io/owning-gateway-name=demo-gateway

Check data plane logs:

    kubectl -n envoy-gateway-system logs <envoy-pod-name> --tail=100

---

## Fourteen. Gateway API Status Check

### 14.1 Check GatewayClass

Execute:

    kubectl get gatewayclass

    kubectl describe gatewayclass eg

Focus on:

    Accepted

---

### 14.2 Check Gateway

Execute:

    kubectl -n gateway-demo get gateway

    kubectl -n gateway-demo describe gateway demo-gateway

Focus on:

    Accepted
    Programmed
    ResolvedRefs

Common normal status:

    Accepted=True
    Programmed=True

---

### 14.3 Check HTTPRoute

Execute:

    kubectl -n gateway-demo get httproute

    kubectl -n gateway-demo describe httproute demo-httproute

Focus on:

    Accepted
    ResolvedRefs

Common normal status:

    Accepted=True
    ResolvedRefs=True

---

### 14.4 Check Backend Service

Execute:

    kubectl -n gateway-demo get svc nginx-gateway-demo

    kubectl -n gateway-demo get endpoints nginx-gateway-demo

Requirement:

    Endpoints must not be empty.

---

## Fifteen. Gateway API and Ingress Coexistence Notes

Gateway API and Ingress can coexist in the same cluster.

Common approaches:

    1. Legacy applications continue using Ingress
    2. New applications gradually adopt Gateway API
    3. Platform team manages Gateway first
    4. Business teams gradually use HTTPRoute
    5. Production environments migrate gradually by business domain and entry layer

Not recommended:

    1. Replace all Ingress without understanding the model
    2. Ingress and Gateway API handling the same domain simultaneously
    3. APIServer VIP and business Gateway entry mixed usage
    4. Production environment directly relying on un-audited public images

---

## Sixteen. Common Issue Troubleshooting

### 16.1 Envoy Gateway Pod ImagePullBackOff

Check Pod:

    kubectl -n envoy-gateway-system get pods -o wide

Check events:

    kubectl -n envoy-gateway-system describe pod <pod-name>

Common causes:

    1. Unable to access docker.io
    2. Unable to pull Envoy Gateway image
    3. Unable to pull Envoy Proxy image
    4. containerd network or proxy anomalies
    5. Internal Harbor not syncing images

Resolution approach:

    1. Check Helm values for image configuration
    2. Sync images to internal Harbor
    3. Modify image address using values file
    4. Re-run helm upgrade

---

### 16.2 Gateway Never Becomes Programmed=True

Check Gateway:

    kubectl -n gateway-demo describe gateway demo-gateway

View GatewayClass:

    kubectl describe gatewayclass eg

View Envoy Gateway logs:

    kubectl -n envoy-gateway-system logs deploy/envoy-gateway --tail=100

Common causes:

    1. GatewayClass controllerName is written incorrectly
    2. Envoy Gateway Controller is not running
    3. EnvoyProxy configuration error
    4. Gateway listener configuration error
    5. Referenced resources cannot be resolved

---

### 16.3 HTTPRoute Not Working

View HTTPRoute:

    kubectl -n gateway-demo describe httproute demo-httproute

Check:

    Accepted
    ResolvedRefs

Common causes:

    1. parentRefs.name is written incorrectly
    2. HTTPRoute and Gateway are not in the same namespace
    3. Gateway allowedRoutes does not allow this HTTPRoute to bind
    4. backendRefs Service name is incorrect
    5. backendRefs Service port is incorrect
    6. hostname mismatch

---

### 16.4 Access Returns 404

Phenomenon:

    curl returns 404

Troubleshoot:

    kubectl -n gateway-demo describe httproute demo-httproute

    kubectl -n gateway-demo describe gateway demo-gateway

Check if the access command includes Host:

    curl -H "Host: demo-gw.ops.local" http://10.0.0.23:${HTTP_NODE_PORT}/

Common causes:

    1. Host mismatch
    2. HTTPRoute hostnames mismatch
    3. path mismatch
    4. Request lacks correct Host header

---

### 16.5 Access Returns 503

Troubleshoot Service:

    kubectl -n gateway-demo get svc nginx-gateway-demo

Troubleshoot Endpoints:

    kubectl -n gateway-demo get endpoints nginx-gateway-demo

Troubleshoot Pod:

    kubectl -n gateway-demo get pods -o wide

Common causes:

    1. Backend Pod is not Running
    2. Service selector mismatch
    3. Endpoints is empty
    4. Service port is incorrect
    5. HTTPRoute backendRefs port is incorrect

---

### 16.6 Cannot Find NodePort

View Envoy Service:

    kubectl -n envoy-gateway-system get svc

Filter by Gateway label:

    kubectl -n envoy-gateway-system get svc \
      -l gateway.envoyproxy.io/owning-gateway-namespace=gateway-demo,gateway.envoyproxy.io/owning-gateway-name=demo-gateway

If Service type is not NodePort, check:

    kubectl -n gateway-demo get envoyproxy nodeport-envoy-proxy -o yaml

    kubectl -n gateway-demo get gateway demo-gateway -o yaml

Focus on confirming:

    Whether Gateway references infrastructure.parametersRef
    Whether EnvoyProxy envoyService.type is NodePort

---

### 16.7 NodePort Access Failure

Check NodePort:

    echo ${HTTP_NODE_PORT}

Check Service:

    kubectl -n envoy-gateway-system describe svc ${ENVOY_SERVICE}

Check kube-proxy:

    kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide

Check IPVS:

    sudo ipvsadm -Ln | grep ${HTTP_NODE_PORT}

Check firewall:

    sudo ufw status

Common causes:

    1. NodePort port is not allowed
    2. kube-proxy anomaly
    3. Envoy Service has no endpoints
    4. Envoy data plane Pod is not Running
    5. Access node network is unreachable

---

## Seventeen, Upgrade and Rollback

### 17.1 View Current Helm Release

Execute:

    helm list -n envoy-gateway-system

Check status:

    helm status eg -n envoy-gateway-system

Check history:

    helm history eg -n envoy-gateway-system

---

### 17.2 Upgrade Envoy Gateway

Before upgrading, it is recommended to back up current values:

    helm get values eg -n envoy-gateway-system -o yaml > envoy-gateway-values-backup.yaml

Upgrade example:

    helm upgrade eg oci://docker.io/envoyproxy/gateway-helm \
      --version ${EG_VERSION} \
      -n envoy-gateway-system

Check status:

    kubectl -n envoy-gateway-system get pods -o wide

---

### 17.3 Rollback Envoy Gateway

Check history:

    helm history eg -n envoy-gateway-system

Rollback: /think

helm rollback eg <REVISION> -n envoy-gateway-system

Check:

    helm status eg -n envoy-gateway-system

---

## Eighteen. Cleanup Test Resources

Delete HTTPRoute:

    kubectl delete -f httproute-demo.yaml

Delete Gateway:

    kubectl delete -f gateway-demo.yaml

Delete GatewayClass:

    kubectl delete -f gatewayclass-eg.yaml

Delete EnvoyProxy:

    kubectl delete -f envoyproxy-nodeport.yaml

Delete test application:

    kubectl -n gateway-demo delete svc nginx-gateway-demo

    kubectl -n gateway-demo delete deployment nginx-gateway-demo

Delete namespace:

    kubectl delete namespace gateway-demo

If you need to uninstall Envoy Gateway:

    helm uninstall eg -n envoy-gateway-system

    kubectl delete namespace envoy-gateway-system

Check CRD:

    kubectl get crd | grep gateway

Notes:

    If there are other Gateway API resources in the cluster, do not arbitrarily delete the Gateway API CRD.
    Deleting the CRD will delete corresponding Gateway, HTTPRoute, etc. custom resources.

---

## Nineteen. Post-Installation Checklist

After installation, execute:

    helm list -n envoy-gateway-system

    kubectl -n envoy-gateway-system get pods -o wide

    kubectl get gatewayclass

    kubectl -n gateway-demo get gateway

    kubectl -n gateway-demo get httproute

    kubectl -n gateway-demo get svc nginx-gateway-demo

    kubectl -n gateway-demo get endpoints nginx-gateway-demo

    kubectl -n envoy-gateway-system get svc

    curl -H "Host: demo-gw.ops.local" http://10.0.0.23:${HTTP_NODE_PORT}/

Requirements:

    1. Envoy Gateway Pod Running
    2. Gateway API CRD is installed
    3. GatewayClass eg exists
    4. Gateway demo-gateway Programmed=True
    5. HTTPRoute demo-httproute Accepted=True
    6. Backend Service Endpoints are not empty
    7. Envoy data plane Service type is NodePort
    8. curl with specified Host can access nginx page

---

## Twenty. Summary

This document completes the basic installation and verification of Gateway API.

Core content:

    1. Install Envoy Gateway
    2. Create EnvoyProxy NodePort configuration
    3. Create GatewayClass
    4. Create Gateway
    5. Create HTTPRoute
    6. Create test application
    7. Verify access using NodePort
    8. Troubleshoot common issues with Gateway, HTTPRoute, NodePort, and backend Service

Gateway API positioning:

    1. GatewayClass binds to specific Controller
    2. Gateway manages ingress listeners
    3. HTTPRoute manages business routing
    4. Service connects to backend Pods

Production recommendations:

    1. Gateway API can coexist with Ingress
    2. Not recommended to directly replace all Ingress at the beginning
    3. Production business entry should be planned with dedicated business VIP or load balancer
    4. Self-hosted clusters can first verify with NodePort
    5. More complete production entry can combine with MetalLB, external LB, F5, HAProxy, or cloud vendor LB
    6. Advanced capabilities like HTTPS, traffic splitting, canary, rate limiting, and authentication are recommended to be documented separately