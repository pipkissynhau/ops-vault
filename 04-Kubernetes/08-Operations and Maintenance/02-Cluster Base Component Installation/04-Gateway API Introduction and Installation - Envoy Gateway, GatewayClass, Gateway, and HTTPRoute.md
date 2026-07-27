# 04-Gateway API Introduction and Installation: Envoy Gateway, GatewayClass, Gateway, and HTTPRoute

Recommended Path:

    04-Kubernetes/08-Operations/02-Cluster Basic Component Installation/04-Gateway API Introduction and Installation: Envoy Gateway, GatewayClass, Gateway, and HTTPRoute.md

Tags:

    #Kubernetes
    #GatewayAPI
    #EnvoyGateway
    #GatewayClass
    #Gateway
    #HTTPRoute
    #Layer 7 Gateway
    #Cluster Basic Component

---

## I. Document Description

This document records the basic installation and verification methods of the Gateway API in a Kubernetes cluster.

This document uses:

    Gateway API
    Envoy Gateway
    GatewayClass
    Gateway
    HTTPRoute
    Verification access via NodePort

Objectives of this document:

    1. Install Envoy Gateway
    2. Create GatewayClass
    3. Create Gateway
    4. Create HTTPRoute
    5. Verify HTTP access via NodePort
    6. Understand the differences between Gateway API and Ingress
    7. Master common troubleshooting methods for Gateway API

Execution Node:

    k8s-master-01

Prerequisites:

    1. The Kubernetes cluster has been deployed.
    2. All Nodes are in the Ready state.
    3. CoreDNS is functioning normally.
    4. CNI is functioning normally.
    5. Helm has been installed.
    6. kubectl can access the cluster successfully.

---

## II. What is Gateway API

Gateway API is a new generation of service gateway and traffic routing API in Kubernetes.

Compared to traditional Ingress, it has a clearer resource model and more distinct responsibilities.

Common Resources:

| Resource | Function |
|---|---|
| GatewayClass | Defines which Gateway Controller will handle it. |
| Gateway | Defines the entry listening port, protocol, domain name, etc. |
| HTTPRoute | Defines how HTTP requests are forwarded to backend Services. |
| ReferenceGrant | Allows cross-namespace resource referencing. |
| TLSRoute | TLS traffic routing. |
| GRPCRoute | gRPC traffic routing.

This document will first focus on the three most basic resources:

    GatewayClass
    Gateway
    HTTPRoute

---

## III. Differences between Gateway API and Ingress

### 3.1 Ingress Model

Common structure of Ingress:

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

    1. Simple model.
    2. Widely used in enterprises.
    3. Mainly designed for Layer 7 HTTP/HTTPS gateways.
    4. Many advanced features rely on Controller annotations.
    5. There are significant differences between different Controller annotations.

---

### 3.2 Gateway API Model

Common structure of Gateway API:

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

    1. Separation of entry infrastructure and business routing.
    2. The Gateway is managed by the platform or operations team.
    3. HTTPRoute can be managed by the business team.
    4. Clearer resource responsibilities.
    5. More suitable for multi-team, multi-tenant, and complex traffic management scenarios.

---

### 3.3 Simple Understanding

It can be understood as follows:

    Ingress:
        One resource contains both entry and routing rules.

    Gateway API:
        The Gateway is responsible for the entry.
        HTTPRoute handles business routing.
        GatewayClass binds to specific Controllers.

---

## IV. Deployment Plan for This Document

This document uses Envoy Gateway as the Gateway API Controller.

| Item | Planning |
|---|---|
| Gateway Controller | Envoy Gateway |
| Installation Method | Helm |
| Envoy Gateway Namespace | envoy-gateway-system |
| Gateway API Test Namespace | gateway-demo |
| GatewayClass Name | eg |
| Gateway Name | demo-gateway |
| HTTPRoute Name | demo-httproute |
| Test Domain Name | demo-gw_ops.local |
| Test Service | nginx-gateway-demo |
    Exposure Method | NodePort |
    Access Port | Automatically generated NodePort |

Note:

    The environment in this document is a self-built kubeadm cluster, which does not have LoadBalancer capabilities by default.
    Therefore, EnvoyProxy is used to expose the data plane Service of Envoy Gateway as a NodePort.
    In a production environment, if MetalLB, SLB, F5, or cloud provider LoadBalancers are available, they can be used instead.

---

## V. Pre-Deployment Checks

### 5.1 Check Cluster Status

Execute:

    kubectl get nodes -    kubectl create namespace gateway-demo

---

### 7.2 Creating an nginx Test Application

Create a Deployment:

    kubectl -n gateway-demo create deployment nginx-gateway-demo --image=nginx:1.25

Scale up:

    kubectl -n gateway-demo scale deployment nginx-gateway-demo --replicas=2

Create a Service:

    kubectl -n gateway-demo expose deployment nginx-gateway-demo \
      --port=80 \
      --target-port=80 \
      --type=ClusterIP

Check:

    kubectl -n gateway-demo get pods -o wide

    kubectl -n gateway-demo get svc

    kubectl -n gateway-demo get endpoints nginx-gateway-demo

Requirements:

    The Pod should be running.
    The Service must exist.
    The Endpoints should not be empty.

---

## VIII. Creating an EnvoyProxy NodePort Configuration

In a self-built Kubernetes cluster using kubeadm, there is generally no default LoadBalancer capability.

Therefore, this document first creates an EnvoyProxy configuration so that the data plane Service created by Envoy Gateway can use the NodePort type.

Create an EnvoyProxy:

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
        This allows the data plane Service created by Envoy Gateway to be exposed using a NodePort.

    envoyDeployment.replicas: 2
        This sets the number of replicas for the Envoy data plane to 2, improving the availability of the entry component.

---

## IX. Creating a GatewayClass

GatewayClass is used to specify which Controller will handle the Gateway.

Create a GatewayClass:

    cat <<EOF > gatewayclass-eg.yaml
    apiVersion: gateway.networking.k8s.io/v1
    kind: GatewayClass
    metadata:
      name: eg
    spec:
      controllerName: gateway.envoyproxy.io/gatewayclass-controller
    EOF

Apply:

    kubectl apply -f gatewayclass-eg.yaml

Check:

    kubectl get gatewayclass

View details:

    kubectl describe gatewayclass eg

Explanation:

    controllerName: gateway.envoyproxy.io/gatewayclass-controller
        This indicates that the GatewayClass is handled by the Envoy Gateway.

---

## X. Creating a Gateway

Gateway defines the specific entry listening configuration.

In this document, an HTTP listener will be created:

    Listening port: 80
    Protocol: HTTP
    Domain name: demo-gw.ops.local

Create a Gateway:

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

Apply:

    kubectl apply -f gateway-demo.yaml

Check:

    kubectl -n gateway-demo get gateway

View details:

    kubectl -n gateway-demo describe gateway demo-gateway

Key points to note:

    Programmed
    Accepted
    ResolvedRefs

Expected result:

    Programmed=True

Explanation:

    infrastructure.parametersRef
        This indicates that the Gateway uses the previously defined EnvoyProxy configuration.

    allowedRoutes.namespaces.from: Same
        This means that only HTTPRoute from the same namespace are allowed to be bound to this Gateway.

---

## XI. Creating an HTTPRoute

HTTPRoute defines how HTTP requests are forwarded to the backend Service.

Create an HTTPRoute:

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

Apply:

    kubectl apply-l gateway.envoyproxy.io/owning-gateway-namespace=gateway-demo,gateway.envoyproxy.io/owning-gateway-name=demo-gateway \
-o jsonpath '{.items[0].metadata.name']

View:

    echo ${ENVOY_SERVICE}

View Service details:

    kubectl -n envoy-gateway-system describe svc ${ENVOY_SERVICE}

Confirm:

    Type: NodePort

 Obtain HTTP NodePort:

    export HTTP_NODE_PORT=$(kubectl -n envoy-gateway-system get svc ${ENVOY_SERVICE} \
      -o jsonpath '{.spec.ports[?(@.port==80)].nodePort}')

View:

    echo ${HTTP NODE_PORT}

Note:

    The NodePort created by Envoy Gateway is automatically assigned by default.
    If a fixed NodePort is required in production, it can be managed separately using methods such as EnvoyProxy Service Patching.

---

## Section Thirteen: Access Verification

### 13.1 Access Using curl with Specified Host

Execute on any machine that can access the NodeIP:

    curl -H "Host: demo-gw.ops.local" http://10.0.0.23:${HTTP_NODE_PORT}/

You can also test other Worker nodes:

    curl -H "Host: demo-gw_ops.local" http://10.0.0.24:${HTTP NODE_PORT}/

    curl -H "Host: demo-gw.ops.local" http://10.0.0.25:${HTTP_NODE_PORT}/

If the default nginx page is returned, it indicates that the Gateway API routing has been successful.

---

### 13.2 Access After Configuring Local hosts

Configure hosts on the client:

    10.0.0.23 demo-gw.ops.local

Access:

    curl http://demo-gw_ops.local:${HTTP_NODE_PORT}/

Note:

    Since this document uses a NodePort, it is necessary to include the NodePort when accessing.
    If an external load balancer is used in front, it can listen on ports 80/443 and then forward requests to the NodePort.

---

### 13.3 Viewing Envoy Gateway Logs

View control plane logs:

    kubectl -n envoy-gateway-system logs deploy/envoy-gateway --tail=100

View Envoy data plane Pods:

    kubectl -n envoy-gateway-system get pods \
      -l gateway.envoyproxy.io/owning-gateway-namespace=gateway-demo,gateway.envoyproxy.io/owning-gateway-name=demo-gateway

View data plane logs:

    kubectl -n envoy-gateway-system logs <envoy-pod-name> --tail=100

---

## Section Fourteen: Checking Gateway API Status

### 14.1 Viewing GatewayClass

Execute:

    kubectl get gatewayclass

    kubectl describe gatewayclass eg

Note:

    Accepted

---

### 14.2 Viewing Gateway

Execute:

    kubectl -n gateway-demo get gateway

    kubectl -n gateway-demo describe gateway demo-gateway

Note:

    Accepted
    Programmed
    ResolvedRefs

Common normal states:

    Accepted=True
    Programmed=True

---

### 14.3 Viewing HTTPRoute

Execute:

    kubectl -n gateway-demo get httproute

    kubectl -n gateway-demo describe httproute demo-httproute

Note:

    Accepted
    ResolvedRefs

Common normal states:

    Accepted=True
    ResolvedRefs=True

---

### 14.4 Viewing Backend Service

Execute:

    kubectl -n gateway-demo get svc nginx-gateway-demo

    kubectl -n gateway-demo get endpoints nginx-gateway-demo

Requirement:

    Endpoints must not be empty.

---

## Section Fifteen: Explanation of Coexistence between Gateway API and Ingress

Gateway API and Ingress can coexist in the same cluster.

Common approaches:

    1. Continue using Ingress for existing services.
    2. Gradually pilot Gateway API for new services.
    3. Let the platform team manage Gateway first.
    4. Have business teams gradually adopt HTTPRoute.
    5. Migrate to production environment based on service domain names and entry layers.

It is not recommended to:

    1. Replace all Ingress without understanding the underlying architecture.
    2. Use both Ingress and Gateway API for the same domain name.
    3. Mix APIServer VIPs with business Gateway entrances.
    4. Rely on unaudited public network images in production.

---

## Section Sixteen: Troubleshooting Common Issues

### 16.1 Envoy Gateway Pod ImagePullBackOff

View Pods:

    kubectl -n envoy-gateway-system get pods -o wide

View Events:

    kubectl -n envoy-gateway-system describe pod <pod-name>

Common### Translate the provided markdown text into English

```markdown
kubectl -n gateway-demo get endpoints nginx-gateway-demo

Troubleshoot Pods:

kubectl -n gateway-demo get pods -o wide

Common causes:

1. The backend Pod is not running.
2. The Service selector does not match.
3. The Endpoints are empty.
4. The Service port is incorrectly specified.
5. The HTTPRoute backendRefs port is incorrectly specified.

---

### 16.6 NodePort Not Found

Check the Envoy Service:

kubectl -n envoy-gateway-system get svc

Filter by Gateway tag:

kubectl -n envoy-gateway-system get svc \
  -l gateway.envoyproxy.io/owning-gateway-namespace=gateway-demo,gateway.envoyproxy.io/owning-gateway-name=demo-gateway

If the Service type is not NodePort, check:

kubectl -n gateway-demo get envoyproxy nodeport-envoy-proxy -o yaml

kubectl -n gateway-demo get gateway demo-gateway -o yaml

Key checks:

- Verify if Gateway references infrastructure.parametersRef.
- Check if envoyService.type in EnvoyProxy is set to NodePort.

---

### 16.7 Inability to Access NodePort

Check the NodePort:

echo ${HTTP_NODE_PORT}

Check the Service:

kubectl -n envoy-gateway-system describe svc ${ENVOY_SERVICE}

Check kube-proxy:

kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide

Check IPVS:

sudo ipvsadm -Ln | grep ${HTTPNODE_PORT}

Check the firewall:

sudo ufw status

Common causes:

1. The NodePort is not open.
2. kube-proxy is malfunctioning.
3. The Envoy Service has no endpoints.
4. The Envoy data plane Pod is not running.
5. The network on the accessing node is down.

---
## Section 17: Upgrade and Rollback

### 17.1 View Current Helm Release

Execute:

helm list -n envoy-gateway-system

Check status:

helm status eg -n envoy-gateway-system

View history:

helm history eg -n envoy-gateway-system

---

### 17.2 Upgrade Envoy Gateway

It is recommended to back up current values before upgrading:

helm get values eg -n envoy-gateway-system -o yaml > envoy-gateway-values-backup.yaml

Example of upgrade:

helm upgrade eg oci://docker.io/envoyproxy/gateway-helm \
  --version ${EG_VERSION} \
  -n envoy-gateway-system

Check status:

kubectl -n envoy-gateway-system get pods -o wide

---

### 17.3 Rollback Envoy Gateway

View history:

helm history eg -n envoy-gateway-system

Rollback:

helm rollback eg <REVISION> -n envoy-gateway-system

Check status:

helm status eg -n envoy-gateway-system
```

## Section 18: Clean Up Test Resources

Delete HTTPRoute:

kubectl delete -f httproute-demo.yaml

Delete Gateway:

kubectl delete -f gateway-demo.yaml

Delete GatewayClass:

kubectl delete -f gatewayclass-eg.yaml

Delete EnvoyProxy:

kubectl delete -f envoyproxy-nodeport.yaml

Delete test applications:

kubectl -n gateway-demo delete svc nginx-gateway-demo

kubectl -n gateway-demo delete deployment nginx-gateway-demo

Delete the namespace:

kubectl delete namespace gateway-demo

If you need to uninstall Envoy Gateway:

helm uninstall eg -n envoy-gateway-system

kubectl delete namespace envoy-gateway-system

Check CRDs:

kubectl get crd | grep gateway

Note:

If there are other Gateway API resources in the cluster, do not delete the Gateway API CRD arbitrarily. Deleting a CRD will also remove corresponding custom resources such as Gateway and HTTPRoute.

---

## Section 19: Post-Installation Checklist

After installation, perform the following checks:

helm list -n envoy-gateway-system

kubectl -n envoy-gateway-system get pods -o wide

kubectl get gatewayclass

kubectl -n gateway-demo get gateway

kubectl -n gateway-demo get httproute

kubectl -n gateway-demo get svc nginx-gateway-demo

kubectl -n gateway-demo get endpoints nginx-gateway-demo

kubectl -n envoy-gateway-system get svc

curl -H "Host: demo-gw.ops.local" http://10.0.0.23:${HTTP_NODE_PORT}/

The following should be true:

1. The Envoy Gateway Pod is running.
2. The Gateway API CRD has been installed.
3. The GatewayClass exists.
4. The Gateway demo-gateway has Programmed set to True.
5. The HTTPRoute demo-httproute has Accepted set to True.
6. The backend Service Endpoints are not empty.
7. The Envoy data plane Service type is NodePort.
8. The curl request to the specified Host should successfully access the nginx page.

---

## Section 20: Summary

This