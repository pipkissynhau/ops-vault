# 03-Service Normal but Business Unreachable: Selector, Endpoints, Ports, and kube-proxy Troubleshooting

Recommended path:

    04-Kubernetes/08-Operations/03-Cluster Basic Troubleshooting/03-Service Normal but Business Unreachable: Selector, Endpoints, Ports, and kube-proxy Troubleshooting.md

Tags:

    #Kubernetes
    #Service
    #Endpoints
    #EndpointSlice
    #kube-proxy
    #IPVS
    #ServiceDiscovery
    #OperationalAccessBarriers
    #ClusterInfrastructureBarriers

---

## I. Document Explanation

This document records troubleshooting methods for scenarios where Kubernetes Service appears normal, Pods are Running, but business access still fails.

This is a very common scenario in Kubernetes operations.

Typical phenomena:

    1. kubectl get svc is normal
    2. kubectl get pod is normal
    3. Pod status is Running
    4. Service has ClusterIP
    5. But access to Service fails
    6. Ingress / Gateway backend returns 502 or 503
    7. Service name access fails inside Pod
    8. curl Service IP has no response
    9. External access fails after NodePort exposure

Document objectives:

    1. Understand the relationship between Service, selector, Endpoints, and EndpointSlice
    2. Determine whether Service is actually connected to backend Pods
    3. Troubleshoot whether selector and Pod label match
    4. Troubleshoot whether port, targetPort, and containerPort are consistent
    5. Determine whether the container's business is actually listening on the port
    6. Troubleshoot kube-proxy / IPVS forwarding rules
    7. Troubleshoot ClusterIP, NodePort, Ingress backend unreachability
    8. Establish a standard troubleshooting path

Applicable scenarios:

    1. Service is normal but access fails
    2. Pod is Running but business is unreachable
    3. Ingress returns 502 / 503
    4. Gateway API backend returns 503
    5. NodePort access fails
    6. Service Endpoints is empty
    7. Service targetPort configuration error
    8. kube-proxy anomaly

---

## II. Service Access Path

Typical Kubernetes Service access path:

    Client
      |
      v
    Service ClusterIP:Port
      |
      v
    kube-proxy forwarding rules
      |
      v
    Endpoints / EndpointSlice
      |
      v
    PodIP:targetPort
      |
      v
    Container business process

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
    Endpoints / EndpointSlice
      |
      v
    PodIP:targetPort
      |
      v
    Container business process

Core judgment:

    Service existence does not guarantee business accessibility.
    Pod Running does not guarantee container business listens on the port.
    Service having ClusterIP does not guarantee it has backend.
    Empty Endpoints means Service cannot forward to backend Pods.

---

## III. Troubleshooting Overview

When Service is normal but business is unreachable, follow this order for troubleshooting:

    1. Confirm Service existence
    2. Check Service selector
    3. Check Pod labels
    4. Confirm selector matches Pod
    5. Check Endpoints / EndpointSlice emptiness
    6. Confirm Service port and targetPort
    7. Confirm container listens on targetPort
    8. curl Service ClusterIP within cluster
    9. Directly curl PodIP
    10. Compare Service access vs Pod access differences
    11. Check kube-proxy / IPVS
    12. Check NodePort / Ingress / Gateway entry layer

Troubleshooting branches:

    Service unreachable
        |
        |-- Endpoints empty
        |       |
        |       |-- selector and label mismatch
        |       |-- Pod not Ready
        |       |-- Pod not in same namespace
        |
        |-- Endpoints not empty
                |
                |-- targetPort error
                |-- container not listening on port
                |-- application listens only on 127.0.0.1
                |-- kube-proxy rule anomaly
                |-- network policy or firewall blocking
                |-- Ingress / Gateway configuration error

---

## IV. Preparation of Test Information

Assume business resources as follows:

    namespace: default
    Deployment: nginx-demo
    Service: nginx-demo
    Service Port: 80
    targetPort: 80
    Pod Label: app=nginx-demo

When troubleshooting, first record: /think

1. namespace  
2. Service Name  
3. Service Type  
4. Service ClusterIP  
5. Service port  
6. targetPort  
7. selector  
8. Pod label  
9. Pod IP  
10. Pod node  
11. Container listening port  

---

## Five. Step One: Check Service  

Execute:  

    kubectl get svc -n default  

Check specified Service:  

    kubectl get svc nginx-demo -n default -o wide  

Check YAML:  

    kubectl get svc nginx-demo -n default -o yaml  

Focus on:  

    spec.type  
    spec.clusterIP  
    spec.ports  
    spec.selector  

Example:  

    spec:  
      clusterIP: 10.96.100.20  
      ports:  
      - port: 80  
        protocol: TCP  
        targetPort: 80  
      selector:  
        app: nginx-demo  
      type: ClusterIP  

Need to confirm:  

    1. Is the Service in the correct namespace  
    2. Does the Service have a ClusterIP  
    3. Is the Service port as expected  
    4. Is the targetPort as expected  
    5. Is the selector as expected  

---

## Six. Step Two: Check Service selector  

Check selector:  

    kubectl get svc nginx-demo -n default -o jsonpath='{.spec.selector}'  

Or:  

    kubectl describe svc nginx-demo -n default  

Example:  

    Selector: app=nginx-demo  

The Service matches Pods via selector.  

If the selector is:  

    app=nginx-demo  

Then the backend Pod must have:  

    app=nginx-demo  

Otherwise, the Service cannot find backend Pods.  

---

## Seven. Step Three: Check Pod label  

Check Pod:  

    kubectl get pods -n default -o wide --show-labels  

Filter label:  

    kubectl get pods -n default -l app=nginx-demo -o wide  

If the command returns no Pods, it means the Service selector matches no Pods.  

Common errors:  

    Service selector:  
        app: nginx-demo  

    Pod label:  
        app: nginx  

In this case, the Service exists but Endpoints will be empty.  

---

## Eight. Step Four: Check Endpoints  

Endpoints are critical for determining if the Service is actually connected to backend Pods.  

Check:  

    kubectl get endpoints nginx-demo -n default  

Or abbreviated:  

    kubectl get ep nginx-demo -n default  

Normal example:  

    NAME         ENDPOINTS                         AGE  
    nginx-demo   10.244.1.10:80,10.244.2.15:80     10m  

Abnormal example:  

    NAME         ENDPOINTS   AGE  
    nginx-demo   <none>      10m  

If Endpoints is `<none>`:  

    The Service definitely has no available backend.  
    At this point, do not continue checking kube-proxy; prioritize checking selector, Pod label, and Pod Ready status.  

Check details:  

    kubectl describe endpoints nginx-demo -n default  

---

## Nine. Step Five: Check EndpointSlice  

Newer Kubernetes versions use EndpointSlice more.  

Check:  

    kubectl get endpointslice -n default  

Filter by Service:  

    kubectl get endpointslice -n default -l kubernetes.io/service-name=nginx-demo  

Check details:  

    kubectl describe endpointslice -n default -l kubernetes.io/service-name=nginx-demo  

Focus on:  

    addresses  
    ports  
    conditions.ready  
    conditions.serving  
    targetRef  

Normally, EndpointSlice should contain Pod IP and port.  

If EndpointSlice is also empty, it means the Service has no backend.  

---

## Ten. Common Causes of Empty Endpoints  

### 10.1 Mismatch between selector and label  

Check Service selector:  

    kubectl get svc nginx-demo -n default -o yaml | grep -A10 selector  

Check Pod label:  

    kubectl get pods -n default --show-labels  

Resolution:  

    1. Modify Service selector  
    2. Modify Pod template labels  
    3. Redeploy Deployment  

Note:  

    Deployment's selector is typically immutable.  
    Do not arbitrarily modify existing Deployment selector.  
    It's more common to fix Service selector or recreate resources.  

---

### 10.2 Pod is not Ready  

Even if Pod is Running, if ReadinessProbe fails, the Pod may not appear in Service Endpoints.  

Check Pod: /think

kubectl get pods -n default -o wide

Check the READY column:

    NAME                          READY   STATUS
    nginx-demo-xxx                 0/1     Running

If READY is 0/1, it indicates the container is not ready.

Check Pod details:

    kubectl describe pod <pod-name> -n default

Check events:

    kubectl get events -n default --sort-by=.lastTimestamp

Common causes:

    1. readinessProbe configuration error
    2. Slow application startup
    3. Incorrect health check path
    4. Incorrect health check port
    5. Application actual exception

---

### 10.3 Pod and Service Not in the Same Namespace

Service can only match Pods in the same namespace via selector.

Check Service namespace:

    kubectl get svc -A | grep nginx-demo

Check Pod namespace:

    kubectl get pods -A | grep nginx-demo

If Service is in default, Pod is in app-prod, Service cannot directly match cross-namespace Pods via selector.

Solutions:

    1. Place Service and Pod in the same namespace
    2. Or use ExternalName / manually configured Endpoints etc. special methods
    3. Ordinary business should not manually bind across namespaces

---

### 10.4 Service Lacks Selector

Some Services have no selector.

Check:

    kubectl get svc nginx-demo -n default -o yaml

If no:

    spec.selector

Kubernetes will not automatically create Endpoints.

This type of Service is typically used for:

    1. ExternalName
    2. Manually configured Endpoints
    3. Connecting to external services
    4. Special traffic forwarding scenarios

Ordinary business Services should typically have a selector.

---

## Eleven, Endpoints Not Empty But Access Fails

If Endpoints is not empty, it indicates Service has found backend Pods.

Next focus on troubleshooting:

    1. Whether targetPort is correct
    2. Whether container listens on corresponding port
    3. Whether application only listens on 127.0.0.1
    4. Whether Service protocol is correct
    5. Whether kube-proxy is normal
    6. Whether network policies block access
    7. Whether application returns exceptions inside Pod

---

## Twelve, Check Service port and targetPort

Check Service:

    kubectl get svc nginx-demo -n default -o yaml

Example:

    ports:
    - port: 80
      targetPort: 8080

Meaning:

    port: 80
        Port Service exposes externally.

    targetPort: 8080
        Port forwarded to Pod.

Access chain:

    ServiceIP:80 -> PodIP:8080

If container actually listens on 80, but targetPort is written as 8080, access will fail.

---

## Thirteen, Check containerPort and actual listening port

### 13.1 Check Pod configuration

Execute:

    kubectl get pod <pod-name> -n default -o yaml | grep -A20 ports

Note:

    containerPort is just a declaration.
    It does not force the application to actually listen on this port.
    Whether it actually listens needs to be checked by entering the container or using network commands.

---

### 13.2 Enter Pod to check listening port

Execute:

    kubectl exec -it <pod-name> -n default -- sh

Check inside container:

    netstat -lntp

If no netstat, try:

    ss -lntp

If the image is minimal, these commands may not exist.

Can use:

    cat /proc/net/tcp

Or temporarily use a debug container.

---

### 13.3 Use temporary debug Pod to access PodIP

Create temporary test Pod:

    kubectl run network-test \
      --image=busybox:1.36 \
      --restart=Never \
      -it --rm -- sh

Access PodIP in test Pod:

    wget -qO- http://<PodIP>:<targetPort>

Example:

    wget -qO- http://10.244.1.10:80

If direct access to PodIP fails, the issue is likely in the backend Pod or network, not Service.

---

## Fourteen, Application Only Listens on 127.0.0.1

Some applications start listening only on:

    127.0.0.1

Instead of:

    0.0.0.0

In this case:

    Container internal local access may be normal
    Other Pods accessing PodIP will fail
    Service access will also fail

Check method:

    kubectl exec -it <pod-name> -n default -- sh

Check listening:

    netstat -lntp

If you see:

    127.0.0.1:8080

It indicates listening only on loopback address.

Should be changed to:

    0.0.0.0:8080

Common configurations:

    server.host=0.0.0.0
    listen 0.0.0.0:8080
    --host 0.0.0.0
    bind-address=0.0.0.0

---

## Fifteen, Test Service Within Cluster

Create temporary test Pod:

    kubectl run curl-test \
      --image=busybox:1.36 \
      --restart=Never \
      -it --rm -- sh

Access Service in test Pod:

    wget -qO- http://nginx-demo.default.svc.cluster.local

Or access short domain name:

    wget -qO- http://nginx-demo

Access ClusterIP:

    wget -qO- http://10.96.100.20

Notes:

    If ClusterIP is reachable but domain name is not, focus on CoreDNS.
    If PodIP is reachable but ClusterIP is not, focus on Service / kube-proxy.
    If PodIP is unreachable, focus on Pod application, port, and CNI network.

---

## Sixteen. Comparing PodIP and ServiceIP

This is a key method to determine the scope of the issue.

### 16.1 Direct Access to PodIP

Get PodIP:

    kubectl get pod -n default -o wide

Access:

    wget -qO- http://<PodIP>:<targetPort>

### 16.2 Access Service ClusterIP

Get ClusterIP:

    kubectl get svc nginx-demo -n default

Access:

    wget -qO- http://<ClusterIP>:<port>

### 16.3 Determine Results

| PodIP Access | ServiceIP Access | Direction |
|---|---|---|
| Not reachable | Not reachable | Backend Pod, application port, CNI |
| Reachable | Not reachable | Service, Endpoints, kube-proxy |
| Reachable | Reachable | Service chain is normal, continue to check Ingress/Gateway/external entry |
| Not reachable | Reachable | Rare, verify test method and target port |

---

## Seventeen. Check kube-proxy

Service ClusterIP forwarding depends on kube-proxy.

### 17.1 Check kube-proxy Pod

Execute:

    kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide

Requirements:

    kube-proxy must be normally Running on each node.

Check logs:

    kubectl -n kube-system logs <kube-proxy-pod-name>

Or:

    kubectl -n kube-system logs -l k8s-app=kube-proxy --tail=100

---

### 17.2 Check kube-proxy Mode

Execute:

    kubectl -n kube-system get cm kube-proxy -o jsonpath='{.data.config\.conf}' | grep -E "mode:|scheduler:|strictARP:"

Common modes:

    iptables
    ipvs

Recommended for this cluster series:

    mode: "ipvs"
    scheduler: "rr"

---

### 17.3 Check IPVS Rules

If kube-proxy uses IPVS mode, execute on the node:

    sudo ipvsadm -Ln

Check specific Service ClusterIP:

    sudo ipvsadm -Ln | grep <ClusterIP>

Example:

    sudo ipvsadm -Ln | grep 10.96.100.20

Check statistics:

    sudo ipvsadm -Ln --stats

If Service exists but no corresponding rule in IPVS, kube-proxy may be abnormal.

---

### 17.4 Check iptables Rules

If using iptables mode, check:

    sudo iptables-save | grep nginx-demo

Or:

    sudo iptables-save | grep <ClusterIP>

Note:

    Troubleshooting commands differ by mode.
    Focus on ipvsadm for IPVS mode.
    Focus on iptables-save for iptables mode.

---

## Eighteen. Troubleshoot NodePort Unreachable

If Service type is NodePort:

    kubectl get svc -n default

Example:

    NAME         TYPE       CLUSTER-IP      PORT(S)
    nginx-demo   NodePort   10.96.100.20    80:30080/TCP

Access chain:

    Client -> NodeIP:30080 -> Service -> Endpoints -> Pod

### 18.1 Check NodePort

Check Service:

    kubectl describe svc nginx-demo -n default

Confirm:

    NodePort
    Port
    TargetPort
    Endpoints

---

### 18.2 Check Node Port

Execute on the node:

    sudo ss -lntp | grep 30080

Note:

    NodePort may not appear as a regular process listening.
    kube-proxy handles traffic via iptables/IPVS rules.
    ss not showing process listening doesn't necessarily indicate an issue.

Recommended check:

    sudo ipvsadm -Ln | grep 30080

---

### 18.3 Check Firewall and Security Group

Execute on the node:

    sudo ufw status

    sudo iptables -L -n

If in a cloud environment, also check:

    Security group
    Firewall
    Network ACL
    External load balancer rules

---

### 18.4 Check externalTrafficPolicy

Check:

    kubectl get svc nginx-demo -n default -o yaml | grep externalTrafficPolicy

If:

    externalTrafficPolicy: Local

Only nodes with backend Pods can normally receive traffic.

If accessing a node without backend Pods, it may be unreachable.

Test:

    kubectl get pod -n default -o wide

Confirm if there are backend Pods on the accessed NodeIP.

# Rules
# Backend
# Events

---

### 19.2 View Backend Service

Run:

    kubectl get svc -n default

    kubectl describe svc <service-name> -n default

Focus on:

    Endpoints

If Endpoints is empty, Ingress will definitely fail to forward to the backend.

---

### 19.3 View ingress-nginx Logs

Run:

    kubectl -n ingress-nginx logs -l app.kubernetes.io/component=controller --tail=100

Common statuses:

    404
        Host or path doesn't match Ingress rules.

    502
        Backend connection failed, possibly due to wrong port, Pod unreachable, or application not listening.

    503
        No available upstream backend, commonly occurs when Endpoints is empty.

---

## Twenty, Gateway API Backend Unreachable Troubleshooting

When Gateway API backend is unreachable, check Service and Endpoints as well.

Check HTTPRoute:

    kubectl get httproute -A

    kubectl describe httproute <name> -n <namespace>

Check Gateway:

    kubectl get gateway -A

    kubectl describe gateway <name> -n <namespace>

Focus on HTTPRoute's:

    Accepted
    ResolvedRefs

If ResolvedRefs=False, it may be due to incorrect Service name or port reference.

Continue checking backend Service:

    kubectl get svc -n <namespace>

    kubectl get endpoints -n <namespace>

---

## Twenty-one, NetworkPolicy Impact

If NetworkPolicy is enabled in the cluster, pod-to-pod communication may be restricted by policies.

Check:

    kubectl get networkpolicy -A

Check specific namespace:

    kubectl get networkpolicy -n default

If there is a default deny policy, it may cause:

    PodIP unreachable
    Service unreachable
    Ingress Controller access to backend unreachable
    Gateway data plane access to backend unreachable

Troubleshooting steps:

    1. Check NetworkPolicy rules
    2. Confirm source Pod label
    3. Confirm target Pod label
    4. Confirm port is allowed
    5. Temporarily relax policy in test environment for verification

Note:

    Do not arbitrarily delete NetworkPolicy in production.
    Confirm business impact scope.

---

## Twenty-two, Protocol Mismatch Issues

Service port defaults to TCP.

If business actually uses UDP, explicitly specify:

    protocol: UDP

Check Service:

    kubectl get svc <svc-name> -n <namespace> -o yaml

Confirm:

    protocol: TCP
    protocol: UDP

Common errors:

    1. DNS service uses UDP but Service is set to TCP
    2. Game, audio/video, gateway service protocol configuration errors
    3. Application uses gRPC but Ingress/Gateway is configured as regular HTTP
    4. HTTPS backend is treated as HTTP forwarding

---

## Twenty-three, Named Port targetPort Issues

Service's targetPort can be a number or a name.

Example:

    ports:
    - port: 80
      targetPort: http

This requires Pod containerPort to have name: http.

Pod example:

    ports:
    - name: http
      containerPort: 8080

If Service targetPort is written as:

    http

But container port lacks name: http, it may cause Endpoints port resolution anomalies.

Check:

    kubectl get svc <svc-name> -n <namespace> -o yaml

    kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A10 ports

Recommendations:

    1. Simple services can directly use numeric targetPort
    2. Multi-port services can use named ports
    3. Named ports must ensure exact name consistency

---

## Twenty-four, Service Multi-port Issues

Multi-port Service must set name for each port.

Example:

    ports:
    - name: http
      port: 80
      targetPort: 8080
    - name: metrics
      port: 9100
      targetPort: 9100

Common issues:

    1. Missing port name
    2. Incorrect targetPort
    3. Ingress backend references wrong port
    4. Prometheus ServiceMonitor references wrong port
    5. Gateway HTTPRoute backendRefs port is wrong

Troubleshoot:

    kubectl get svc <svc-name> -n <namespace> -o yaml

    kubectl describe svc <svc-name> -n <namespace>

---

## Twenty-five, Common Issues Quick Check

| Phenomenon | Common Causes | Priority Checks |
|---|---|---|
| Service has ClusterIP but is inaccessible | Endpoints is empty | kubectl get endpoints |
| Endpoints is empty | selector and label mismatch | svc selector / pod labels |
| Pod Running but not in Endpoints | ReadinessProbe failure | pod READY / describe pod |
| PodIP is unreachable | Application not listening, CNI issues | curl PodIP / check ports inside container |
| PodIP reachable but ServiceIP unreachable | kube-proxy anomaly | kube-proxy / ipvsadm |
| Ingress 503 | Service has no Endpoints | describe ingress / svc endpoints |
| Ingress 502 | Backend port error or connection failure | targetPort / Pod listening |
| NodePort unreachable | Firewall, kube-proxy, externalTrafficPolicy | ipvsadm / ufw / Pod distribution |
| Domain unreachable but ClusterIP reachable | DNS issue | CoreDNS |
| Service targetPort is name | containerPort name mismatch | svc yaml / pod yaml |

---

## Twenty-six, Standard Troubleshooting Command List

### 26.1 Service Layer

    kubectl get svc -n <namespace>

    kubectl describe svc <svc-name> -n <namespace>

    kubectl get svc <svc-name> -n <namespace> -o yaml

---

### 26.2 Endpoints Layer

    kubectl get endpoints <svc-name> -n <namespace>

    kubectl describe endpoints <svc-name> -n <namespace>

    kubectl get endpointslice -n <namespace> -l kubernetes.io/service-name=<svc-name>

    kubectl describe endpointslice -n <namespace> -l kubernetes.io/service-name=<svc-name>

---

### 26.3 Pod Layer

    kubectl get pods -n <namespace> -o wide --show-labels

    kubectl describe pod <pod-name> -n <namespace>

    kubectl logs <pod-name> -n <namespace>

    kubectl exec -it <pod-name> -n <namespace> -- sh

---

### 26.4 Testing Access

    kubectl run curl-test --image=busybox:1.36 --restart=Never -it --rm -- sh

    wget -qO- http://<ServiceName>.<Namespace>.svc.cluster.local

    wget -qO- http://<ClusterIP>:<Port>

    wget -qO- http://<PodIP>:<TargetPort>

---

### 26.5 kube-proxy / IPVS

    kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide

    kubectl -n kube-system logs -l k8s-app=kube-proxy --tail=100

    kubectl -n kube-system get cm kube-proxy -o jsonpath='{.data.config\.conf}' | grep -E "mode:|scheduler:|strictARP:"

    sudo ipvsadm -Ln

    sudo ipvsadm -Ln --stats

---

### 26.6 Ingress / Gateway

    kubectl get ingress -A

    kubectl describe ingress <name> -n <namespace>

    kubectl -n ingress-nginx logs -l app.kubernetes.io/component=controller --tail=100

    kubectl get gateway -A

    kubectl get httproute -A

    kubectl describe httproute <name> -n <namespace>

---

## Twenty-seven, Recommended Troubleshooting Path

### 27.1 Service Inaccessibility

Execution order:

    kubectl get svc <svc-name> -n <namespace>

    kubectl describe svc <svc-name> -n <namespace>

    kubectl get endpoints <svc-name> -n <namespace>

    kubectl get pods -n <namespace> --show-labels

    kubectl get pods -n <namespace> -o wide

    curl / wget direct access to PodIP

    curl / wget access to ServiceIP

    Check kube-proxy / IPVS

---

### 27.2 Ingress 502 / 503

Execution order:

    kubectl describe ingress <ingress-name> -n <namespace>

    kubectl describe svc <svc-name> -n <namespace>

    kubectl get endpoints <svc-name> -n <namespace>

    kubectl get pods -n <namespace> -o wide

    kubectl -n ingress-nginx logs -l app.kubernetes.io/component=controller --tail=100

---

### 27.3 NodePort Inaccessibility

Execution order:

kubectl describe svc <svc-name> -n <namespace>

kubectl get endpoints <svc-name> -n <namespace>

kubectl get pods -n <namespace> -o wide

sudo ipvsadm -Ln | grep <NodePort>

sudo ufw status

curl http://<NodeIP>:<NodePort>

---

## Twenty-Eight: Summary

When a Service is functioning normally but the business application is unreachable, do not only check if the Service exists or if the Pod is Running.

Core judgment points:

    1. Is the Service selector correct?
    2. Do the Pod labels match?
    3. Is the Endpoints empty?
    4. Are the EndpointSlice normal?
    5. Is the targetPort correct?
    6. Is the container actually listening on the port?
    7. Is the application listening on 0.0.0.0?
    8. Can the PodIP be accessed?
    9. Can the ClusterIP be accessed?
    10. Is kube-proxy functioning normally?
    11. Are the IPVS / iptables rules present?
    12. Do the Ingress / Gateway reference the correct Service and port?

Most important commands:

    kubectl describe svc <svc-name> -n <namespace>

    kubectl get endpoints <svc-name> -n <namespace>

    kubectl get pods -n <namespace> --show-labels

Troubleshooting experience:

    1. If Endpoints are empty, prioritize checking the selector, labels, and ReadinessProbe.
    2. If PodIP is unreachable, prioritize checking the application listening, container port, and CNI.
    3. If PodIP is reachable but ServiceIP is not, prioritize checking kube-proxy.
    4. If ServiceIP is reachable but Ingress is not, prioritize checking Ingress rules and Controller logs.
    5. If NodePort is unreachable, prioritize checking the firewall, kube-proxy, and externalTrafficPolicy.
    6. Do not assume the Service exists means the business is definitely accessible.