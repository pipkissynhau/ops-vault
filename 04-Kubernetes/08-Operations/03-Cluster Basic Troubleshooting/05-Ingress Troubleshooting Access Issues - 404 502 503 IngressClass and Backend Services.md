# 05-Ingress Access Anomalies Troubleshooting: 404, 502, 503, IngressClass and Backend Services

Recommended Path:

    04-Kubernetes/08-Operations/03-Cluster Basic Troubleshooting/05-Ingress Access Anomalies Troubleshooting: 404, 502, 503, IngressClass and Backend Services.md

Tags:

    #Kubernetes
    #Ingress
    #ingress-nginx
    #IngressClass
    #Nginx
    #Service
    #Endpoints
    #HttpBarrier
    #ClusterInfrastructureBarriers

---

## I. Document Description

This document records basic troubleshooting methods for Ingress access anomalies in Kubernetes clusters.

Ingress is a commonly used seven-layer entry resource in Kubernetes, typically used with components like ingress-nginx, Traefik, HAProxy Ingress, and cloud vendor Ingress Controllers.

This document primarily focuses on troubleshooting with ingress-nginx.

Common issues:

    1. Accessing Ingress returns 404
    2. Accessing Ingress returns 502
    3. Accessing Ingress returns 503
    4. Domain access fails
    5. NodePort can be accessed but Ingress fails
    6. IngressClass mismatch
    7. Service backend Endpoints are empty
    8. targetPort configuration error
    9. ingress-nginx-controller Pod anomaly
    10. TLS certificate anomaly
    11. HTTP/HTTPS configuration mismatch

Document objectives:

    1. Understand Ingress access flow
    2. Be able to judge the common meanings of 404, 502, 503
    3. Be able to troubleshoot IngressClass
    4. Be able to troubleshoot Host and Path matching
    5. Be able to troubleshoot backend Service
    6. Be able to troubleshoot Endpoints
    7. Be able to troubleshoot ingress-nginx Controller
    8. Be able to view ingress-nginx logs
    9. Be able to troubleshoot TLS certificate issues
    10. Establish a standard Ingress troubleshooting path

Applicable scenarios:

    1. kubeadm self-hosted cluster
    2. ingress-nginx NodePort exposure
    3. ingress-nginx LoadBalancer exposure
    4. Private Kubernetes cluster
    5. Business domain access anomalies
    6. Ingress backend service anomalies
    7. HTTPS access anomalies

---

## II. Ingress Access Flow

Typical Ingress access flow:

    User / Client
        |
        v
    Domain DNS resolution
        |
        v
    External entry IP / VIP / NodeIP
        |
        v
    ingress-nginx-controller Service
        |
        v
    ingress-nginx-controller Pod
        |
        v
    Ingress rule matching Host / Path
        |
        v
    Backend Service
        |
        v
    Endpoints / EndpointSlice
        |
        v
    PodIP:targetPort
        |
        v
    Container business process

If using NodePort:

    User
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
    Ingress -> Service -> Pod

If using MetalLB LoadBalancer:

    User
      |
      v
    MetalLB VIP:80 / 443
      |
      v
    ingress-nginx-controller Service(type=LoadBalancer)
      |
      v
    ingress-nginx-controller Pod
      |
      v
    Ingress -> Service -> Pod

Core judgment:

    Ingress itself is just a rule.
    The actual traffic receiver is the Ingress Controller.
    The backend still ultimately depends on Service and Endpoints.

---

## III. Common Status Code Meanings

### 3.1 404 Not Found

Common meanings:

    Request reached ingress-nginx
    But no matching Ingress rule was found

Common causes:

    1. Host mismatch
    2. Path mismatch
    3. ingressClassName mismatch
    4. Ingress not recognized by controller
    5. Request lacks correct Host header
    6. Accessed default backend

Typical error:

    curl http://10.0.0.23:30080/

But Ingress rule requires:

    host: demo.ops.local

Correct testing method:

    curl -H "Host: demo.ops.local" http://10.0.0.23:30080/

---

### 3.2 502 Bad Gateway

Common meanings:

    Ingress rule matched
    But ingress-nginx failed to connect backend

Common causes:

    1. Service targetPort written incorrectly
    2. Container in Pod not listening on port
    3. Application only listens on 127.0.0.1
    4. Backend connection rejected
    5. Backend protocol mismatch
    6. Backend application exited abnormally
    7. NetworkPolicy blocked
    8. CNI network anomaly

Simple understanding:

    502 is moreDiverse "backend connection failure".

---

### 3.3 503 Service Temporarily Unavailable

Common meanings:

    Ingress rule matched
    But no available upstream backend

Common causes: /think

1. Service Endpoints is empty  
2. Pod is not Ready  
3. Service selector does not match Pod label  
4. Backend Service does not exist  
5. Ingress backend references the wrong Service  
6. Ingress backend references the wrong port  

Simple understanding:  

    503 is more inclined toward "no available backend".  

---

## Four. Overall Troubleshooting Approach  

Ingress anomaly troubleshooting is recommended to follow this order:  

    1. Confirm the access entry is correct  
    2. Confirm ingress-nginx-controller is normal  
    3. Confirm Ingress exists  
    4. Confirm IngressClass matches  
    5. Confirm Host matches  
    6. Confirm Path matches  
    7. Confirm backend Service exists  
    8. Confirm Service Endpoints is empty  
    9. Confirm Service port / targetPort is correct  
    10. Confirm Pod is Ready  
    11. Directly access Service / PodIP for comparison  
    12. Check ingress-nginx logs  
    13. Troubleshoot TLS / HTTPS  
    14. Troubleshoot DNS / external LB / firewall  

Troubleshooting branches:  

    Ingress access anomaly  
        |  
        |-- 404  
        |     |  
        |     |-- Host does not match  
        |     |-- Path does not match  
        |     |-- IngressClass does not match  
        |     |-- Request lacks Host header  
        |  
        |-- 502  
        |     |  
        |     |-- targetPort is incorrect  
        |     |-- Pod is not listening on the port  
        |     |-- Application only listens on 127.0.0.1  
        |     |-- Backend protocol error  
        |  
        |-- 503  
              |  
              |-- Service does not exist  
              |-- Endpoints is empty  
              |-- Pod is not Ready  
              |-- Selector does not match  

---

## Five. Preparation of Troubleshooting Information  

Before troubleshooting, record the following information:  

    1. Access domain  
    2. Access protocol: http / https  
    3. Access entry: NodePort / LoadBalancer / external Nginx / F5 / SLB  
    4. Ingress namespace  
    5. Ingress name  
    6. IngressClass  
    7. backend Service name  
    8. backend Service port  
    9. backend Pod label  
    10. ingress-nginx namespace  
    11. ingress-nginx exposure method  
    12. Specific error status code  

Example:  

    Domain: demo.ops.local  
    Access entry: 10.0.0.23:30080  
    Ingress: demo/nginx-demo  
    IngressClass: nginx  
    Service: demo/nginx-demo:80  
    Pod: nginx-demo  
    Status code: 404 / 502 / 503  

---

## Six. First Step: Confirm ingress-nginx-controller Status  

Check Pod:  

    kubectl -n ingress-nginx get pods -o wide  

Expected:  

    ingress-nginx-controller Pod Running  

If there are multiple replicas:  

    All replicas should be Running  
    READY should be 1/1 or match the actual container count  

Check Deployment:  

    kubectl -n ingress-nginx get deploy  

Check detailed information:  

    kubectl -n ingress-nginx describe deploy ingress-nginx-controller  

If controller Pod is abnormal, troubleshoot the controller itself first:  

    ImagePullBackOff  
    CrashLoopBackOff  
    Pending  
    Node NotReady  
    Configuration error  
    Admission webhook anomaly  

---

## Seven. Second Step: Confirm ingress-nginx Service Exposure Method  

Check Service:  

    kubectl -n ingress-nginx get svc  

Common types:  

    NodePort  
    LoadBalancer  
    ClusterIP  

NodePort example:  

    NAME                       TYPE       CLUSTER-IP     PORT(S)  
    ingress-nginx-controller   NodePort   10.96.10.20    80:30080/TCP,443:30443/TCP  

LoadBalancer example:  

    NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP  
    ingress-nginx-controller   LoadBalancer   10.96.10.20    10.0.0.102  

Check details:  

    kubectl -n ingress-nginx describe svc ingress-nginx-controller  

Confirm:  

    1. Service exists  
    2. Type matches expectation  
    3. HTTP port is correct  
    4. HTTPS port is correct  
    5. NodePort is correct  
    6. EXTERNAL-IP is successfully assigned  
    7. Endpoints exist  

If ingress-nginx-controller Service itself has no Endpoints, it indicates the controller Pod is not properly attached to the Service backend.  

---

## Eight. Third Step: Confirm Entry Can Access ingress-nginx

### 8.1 Testing with NodePort

Assume HTTP NodePort is 30080:

    curl -I http://10.0.0.23:30080/

If returns:

    HTTP/1.1 404 Not Found

This typically indicates:

    The request has reached ingress-nginx
    But no Ingress rule matches

This is not a bad thing, it means the ingress chain at least reaches the controller.

If connection fails:

    Connection refused
    Connection timed out
    No route to host

Prioritize checking:

    1. Whether NodePort is correct
    2. Whether ingress-nginx Service is normal
    3. Whether ingress-nginx Pod is Running
    4. Whether kube-proxy is normal
    5. Whether node firewall allows traffic
    6. Whether security group or network ACL allows traffic

---

### 8.2 Testing with LoadBalancer

Assume EXTERNAL-IP is 10.0.0.102:

    curl -I http://10.0.0.102/

If returns 404, it indicates the request has reached ingress-nginx.

If access fails, check:

    1. Whether MetalLB is functioning properly
    2. Whether EXTERNAL-IP is assigned
    3. Whether network from client to EXTERNAL-IP is reachable
    4. Whether firewall allows traffic
    5. Whether ingress-nginx Service is normal

---

## Nine. Fourth Step: Check Ingress Resources

Check all Ingress:

    kubectl get ingress -A

Check specific namespace:

    kubectl -n demo get ingress

Check details:

    kubectl -n demo describe ingress nginx-demo

Check YAML:

    kubectl -n demo get ingress nginx-demo -o yaml

Focus on:

    1. ingressClassName
    2. rules.host
    3. paths.path
    4. paths.pathType
    5. backend.service.name
    6. backend.service.port.number/name
    7. tls.secretName
    8. Events

---

## Ten. Fifth Step: Check IngressClass

### 10.1 Check IngressClass

Execute:

    kubectl get ingressclass

Example:

    NAME    CONTROLLER             PARAMETERS
    nginx   k8s.io/ingress-nginx   <none>

Check details:

    kubectl describe ingressclass nginx

---

### 10.2 Check Ingress Using Class

Execute:

    kubectl -n demo get ingress nginx-demo -o yaml | grep ingressClassName

Expect:

    ingressClassName: nginx

If ingress-nginx is configured with:

    watchIngressWithoutClass: false

Then Ingress without ingressClassName will not be processed.

---

### 10.3 Common IngressClass Issues

Common errors:

    1. Ingress did not write ingressClassName
    2. ingressClassName is written incorrectly
    3. Cluster has no corresponding IngressClass
    4. Multiple Ingress Controllers mixed usage
    5. Old annotation and new fields mixed usage

Old syntax may use annotation:

    kubernetes.io/ingress.class: nginx

New version recommends:

    spec.ingressClassName: nginx

Handling method:

    Modify Ingress:

        ingressClassName: nginx

Reapply:

    kubectl apply -f ingress.yaml

---

## Eleven. 404 Troubleshooting: Host Mismatch

### 11.1 Check Ingress Host

Execute:

    kubectl -n demo get ingress nginx-demo

Or:

    kubectl -n demo get ingress nginx-demo -o yaml | grep -A5 host

Example:

    rules:
    - host: demo.ops.local

---

### 11.2 curl Must Include Host

If accessing NodePort:

    curl http://10.0.0.23:30080/

Request Host is actually:

    10.0.0.23

Does not match:

    demo.ops.local

Correct way:

    curl -H "Host: demo.ops.local" http://10.0.0.23:30080/

If using local hosts:

    10.0.0.23 demo.ops.local

Access:

    curl http://demo.ops.local:30080/

---

### 11.3 Browser Access Note

If using NodePort, browser access must include port:

    http://demo.ops.local:30080/

If using LoadBalancer, usually can access directly:

    http://demo.ops.local/

Provided:

    demo.ops.local is resolved to LoadBalancer IP

---

## Twelve. 404 Troubleshooting: Path Mismatch

Check Ingress path:

    kubectl -n demo get ingress nginx-demo -o yaml | grep -A20 paths

Example:

    path: /
    pathType: Prefix

Common pathType:

    Prefix
    Exact
    ImplementationSpecific

### 12.1 Prefix

Configuration:

    path: /api
    pathType: Prefix

Matches:

    /api
    /api/
    /api/v1

### 12.2 Exact

Configuration:

    path: /api
    pathType: Exact

Only matches:

    /api

Does not match:

    /api/
    /api/v1

Common issues:

    1. path is written as /api, but actual access is /
    2. pathType is Exact, but subpaths do not match
    3. Multiple Ingress rules have conflicting paths
    4. Missing or incorrect rewrite rules

---

## ThirteenI don't know.404 Troubleshooting: Ingress not recognized by controller

Check ingress-nginx logs:

    kubectl -n ingress-nginx logs -l app.kubernetes.io/component=controller --tail=200

You can also check if the controller has loaded the Ingress:

    kubectl -n demo describe ingress nginx-demo

Focus on the Events section.

Common causes:

    1. IngressClass mismatch
    2. Ingress YAML format error
    3. Backend Service does not exist
    4. Admission webhook rejected
    5. ingress-nginx controller permission anomaly

---

## FourteenI don't know.503 Troubleshooting: Service does not exist

Check Ingress backend:

    kubectl -n demo get ingress nginx-demo -o yaml | grep -A20 backend

Example:

    backend:
      service:
        name: nginx-demo
        port:
          number: 80

Check Service:

    kubectl -n demo get svc nginx-demo

If it does not exist:

    Error from server (NotFound): services "nginx-demo" not found

Resolution:

    1. Create Service
    2. Correct Ingress backend service name
    3. Confirm namespace is correct

Note:

    The Service referenced by Ingress must be in the same namespace.
    Ordinary Ingress backend does not reference Service across namespaces.

---

## FifteenI don't know.503 Troubleshooting: Endpoints are empty

Check Service:

    kubectl -n demo describe svc nginx-demo

Focus on:

    Endpoints

If it is:

    Endpoints: <none>

This indicates the Service has no available backend.

Continue checking:

    kubectl -n demo get endpoints nginx-demo

    kubectl -n demo get endpointslice -l kubernetes.io/service-name=nginx-demo

Common causes for empty Endpoints:

    1. Service selector does not match Pod label
    2. Pod is not Ready
    3. ReadinessProbe failed
    4. Pod is not in the same namespace
    5. Service has no selector
    6. Pod has restarted or is abnormal

---

### 15.1 Check selector and label

Check Service selector:

    kubectl -n demo get svc nginx-demo -o yaml | grep -A10 selector

Check Pod label:

    kubectl -n demo get pods --show-labels

Query by selector:

    kubectl -n demo get pods -l app=nginx-demo -o wide

If no Pod is found, the selector does not match.

---

### 15.2 Check Pod Ready

Check Pod:

    kubectl -n demo get pods -o wide

Focus on READY:

    0/1
    1/1

If it is 0/1:

    kubectl -n demo describe pod <pod-name>

Check for:

    Readiness probe failed
    connection refused
    HTTP probe failed
    timeout

Resolution:

    1. Correct readinessProbe
    2. Confirm the actual health check path for the application
    3. Confirm the probe port is correct
    4. Extend initialDelaySeconds
    5. Fix application anomalies

---

## SixteenI don't know.502 Troubleshooting: targetPort error

Check Service:

    kubectl -n demo get svc nginx-demo -o yaml

Example:

    ports:
    - port: 80
      targetPort: 8080

Meaning:

    Ingress -> Service:80 -> PodIP:8080

If the container actually listens on 80, but targetPort is written as 8080, it may result in 502.

Check Pod configuration:

    kubectl -n demo get pod <pod-name> -o yaml | grep -A20 ports

Note:

    containerPort is just a declaration.
    Whether the container is actually listening needs to be checked inside the container.

---

## SeventeenI don't know.502 Troubleshooting: Container is not listening on port

Enter Pod:

    kubectl -n demo exec -it <pod-name> -- sh

Check for listening inside the container:

    netstat -lntp

Or:

    ss -lntp

If the image does not have the command, use a temporary debug Pod to test PodIP.

Check PodIP:

    kubectl -n demo get pod -o wide

Temporary test:

    kubectl run curl-test \
      --image=busybox:1.36 \
      --restart=Never \
      -it --rm -- sh

Execute inside the test Pod:

    wget -qO- http://<PodIP>:<targetPort>

If PodIP:targetPort is unreachable, the Ingress will definitely be unreachable.

## EighteenI don't know.502 Troubleshooting: Application Listens Only on 127.0.0.1

Enter the container to check listening status:

    netstat -lntp

If you see:

    127.0.0.1:8080

It indicates the application only listens on the container's local loopback address.

Other Pods cannot access this port via PodIP.

Should be changed to listen on:

    0.0.0.0:8080

Common configurations:

    --host 0.0.0.0
    server.address=0.0.0.0
    listen 0.0.0.0:8080
    bind-address=0.0.0.0

---

## NineteenI don't know.502 Troubleshooting: HTTP/HTTPS Backend Protocol Mismatch

Some backend services only accept HTTPS, but Ingress defaults to forwarding via HTTP.

Common symptoms:

    502
    upstream prematurely closed connection
    SSL_do_handshake failed
    backend protocol error

If the backend is HTTPS, configure the ingress-nginx annotation:

    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"

Example:

    metadata:
      annotations:
        nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"

Note:

    This is an annotation for ingress-nginx.
    Different Ingress Controllers have different configuration methods.

---

## TwentyI don't know.Check ingress-nginx Logs

View controller logs:

    kubectl -n ingress-nginx logs -l app.kubernetes.io/component=controller --tail=200

Real-time viewing:

    kubectl -n ingress-nginx logs -l app.kubernetes.io/component=controller -f

If there are multiple controller Pods, check a specific Pod:

    kubectl -n ingress-nginx get pods -o wide

    kubectl -n ingress-nginx logs <controller-pod-name> --tail=200

Focus on the following in logs:

    1. Request status codes
    2. Upstream address
    3. Upstream response time
    4. no endpoints available
    5. service not found
    6. upstream timed out
    7. connect() failed
    8. SSL handshake failed

Common log directions:

    no endpoints available
        Backend Service is empty.

    service not found
        The Service referenced by Ingress does not exist.

    connect() failed
        Backend connection failed, commonly due to targetPort errors or application not listening.

    upstream timed out
        Backend response timeout, possibly due to slow business, network blockage, or insufficient timeout.

---

## Twenty-oneI don't know.View ingress-nginx Dynamic Configuration (Optional)

ingress-nginx generates Nginx configuration based on Ingress.

Enter the controller Pod:

    kubectl -n ingress-nginx exec -it <controller-pod-name> -- sh

View Nginx configuration:

    nginx -T

Or filter by domain:

    nginx -T | grep -n "demo.ops.local" -A30

Note:

    This operation is suitable for advanced troubleshooting.
    Basic troubleshooting should prioritize checking Ingress, Service, Endpoints, and logs.

---

## Twenty-twoI don't know.HTTPS/TLS Abnormality Troubleshooting

### 22.1 Check Ingress TLS Configuration

Execute:

    kubectl -n demo get ingress nginx-demo -o yaml | grep -A10 tls

Example:

    tls:
    - hosts:
      - demo.ops.local
      secretName: demo-tls

Confirm:

    1. tls.hosts correctness
    2. secretName correctness
    3. Secret existence
    4. Secret type is kubernetes.io/tls

---

### 22.2 Check TLS Secret

View Secret:

    kubectl -n demo get secret demo-tls

Check type:

    kubectl -n demo get secret demo-tls -o yaml | grep type

Expected:

    type: kubernetes.io/tls

Check fields:

    kubectl -n demo get secret demo-tls -o jsonpath='{.data}' | head

Should include:

    tls.crt
    tls.key

---

### 22.3 View Certificate Content

Check validity period:

    kubectl -n demo get secret demo-tls \
      -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -dates

Check SAN:

    kubectl -n demo get secret demo-tls \
      -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -text | grep -A2 "Subject Alternative Name"

Confirm the domain is in the certificate SAN.

---

### 22.4 curl HTTPS Test

Self-signed certificate test:

    curl -k -H "Host: demo.ops.local" https://10.0.0.23:30443/

Production certificate test:

    curl -H "Host: demo.ops.local" https://10.0.0.23:30443/

If the certificate is not trusted, use -k temporarily for validation, but do not rely on -k in production.

### 22.5 cert-manager Related Checks

If certificates are managed by cert-manager:

    kubectl -n demo get certificate

    kubectl -n demo describe certificate <certificate-name>

    kubectl -n cert-manager get pods -o wide

    kubectl -n cert-manager logs deploy/cert-manager --tail=100

Common Issues:

    1. Certificate not Ready
    2. ClusterIssuer does not exist
    3. DNS-01 / HTTP-01 validation failure
    4. Secret not generated
    5. Secret name and Ingress reference mismatch

---

## Twenty-Three, Ingress Annotation Issues

ingress-nginx's many advanced capabilities are controlled through annotations.

Check Ingress annotations:

    kubectl -n demo get ingress nginx-demo -o yaml | grep -A30 annotations

Common Annotations:

    nginx.ingress.kubernetes.io/rewrite-target
    nginx.ingress.kubernetes.io/backend-protocol
    nginx.ingress.kubernetes.io/proxy-body-size
    nginx.ingress.kubernetes.io/proxy-read-timeout
    nginx.ingress.kubernetes.io/proxy-send-timeout
    nginx.ingress.kubernetes.io/ssl-redirect
    nginx.ingress.kubernetes.io/use-regex

Common Issues:

    1. rewrite-target configuration error causing path forwarding anomalies
    2. backend-protocol not set causing HTTPS backend 502
    3. proxy-body-size too small causing upload failure
    4. proxy-read-timeout too small causing long request timeout
    5. ssl-redirect causing HTTP automatic redirect to HTTPS
    6. use-regex mismatch with path writing format

Troubleshooting Recommendations:

    First use the simplest Ingress to verify the chain.
    Then gradually add rewrite, timeout, HTTPS, and other advanced configurations.

---

## Twenty-Four, Rewrite Path Issues

Common Scenarios:

    User accesses /api
    Backend application only recognizes /

Without rewrite, the backend may receive /api and return 404.

Example Configuration:

    metadata:
      annotations:
        nginx.ingress.kubernetes.io/rewrite-target: /
    spec:
      rules:
      - host: demo.ops.local
        http:
          paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: nginx-demo
                port:
                  number: 80

Note:

    rewrite is a capability of ingress-nginx.
    Behavior varies with different versions and pathType, needs actual testing.
    Rewrite configuration errors can easily cause access path anomalies.

---

## Twenty-Five, Upload File 413 Issue

Phenomenon:

    413 Request Entity Too Large

Common Causes:

    ingress-nginx default request body size limit does not meet business upload requirements.

Resolution:

    metadata:
      annotations:
        nginx.ingress.kubernetes.io/proxy-body-size: "100m"

Check Logs:

    kubectl -n ingress-nginx logs -l app.kubernetes.io/component=controller --tail=100

---

## Twenty-Six, Request Timeout Issue

Phenomenon:

    504 Gateway Timeout
    upstream timed out

Common Causes:

    1. Backend processing is slow
    2. ingress-nginx timeout time is too short
    3. Backend service is stuck
    4. Network link anomaly

Common Annotations:

    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"

Note:

    Do not blindly set timeout time to a very large value.
    Should also troubleshoot backend business performance simultaneously.

---

## Twenty-Seven, DNS Resolution Issue

If browser access to domain is not working, first confirm domain resolution.

Check Resolution:

    nslookup demo.ops.local

    dig demo.ops.local

For local testing, configure hosts:

    10.0.0.23 demo.ops.local

Or LoadBalancer:

    10.0.0.102 demo.ops.local

Note:

    DNS resolution directs traffic to where it resolves.
    If resolved to wrong IP, Ingress configuration is correct but still inaccessible.

---

## Twenty-Eight, External Load Balancer Issue

If Ingress has:

    F5
    Nginx
    HAProxy
    SLB
    ELB
    Firewall
    WAF

Need to confirm:

    1. External LB listens on 80 / 443
    2. External LB forwards to correct NodePort
    3. Health check is normal
    4. Host header is preserved
    5. HTTPS termination is done
    6. Duplicate TLS termination
    7. Path is rewritten
    8. Request headers are intercepted

Troubleshooting Approach: /think

1. Bypass external LB, directly curl NodePort  
2. Directly curl ingress-nginx LoadBalancer IP  
3. Compare external access and internal access results  
4. Check external LB logs  

---

## 29. NetworkPolicy Impact  

If NetworkPolicy is enabled, traffic from ingress-nginx-controller to business Pods may be blocked.  

Check:  

    kubectl get networkpolicy -A  

Check backend namespace:  

    kubectl -n demo get networkpolicy  

Need to allow:  

    ingress-nginx namespace controller Pod  
    access business Pods in demo namespace  

Common phenomena:  

    1. Service Endpoints is not empty  
    2. PodIP port exists  
    3. But ingress-nginx times out accessing backend  
    4. Logs show upstream timed out  

Production note:  

    Do not delete NetworkPolicy arbitrarily.  
    Should precisely allow based on business access relationships.  

---

## 30. Service and Ingress Port Reference Issues  

Ingress backend can reference Service port number or port name.  

Service example:  

    ports:  
    - name: http  
      port: 80  
      targetPort: 8080  

Ingress can write:  

    service:  
      name: nginx-demo  
      port:  
        number: 80  

Or:  

    service:  
      name: nginx-demo  
      port:  
        name: http  

Common errors:  

    1. Ingress writes number: 8080, but Service port is 80  
    2. Ingress writes name: web, but Service port name is http  
    3. Service targetPort is wrong  
    4. Multi-port Service references wrong port  

Troubleshoot:  

    kubectl -n demo get svc nginx-demo -o yaml  

    kubectl -n demo get ingress nginx-demo -o yaml  

---

## 31. Ingress and Service Namespace Issues  

The Service referenced by Ingress backend must be in the same namespace.  

Example:  

    Ingress in namespace demo  
    Service must also be in namespace demo  

Check:  

    kubectl get ingress -A  

    kubectl get svc -A  

If Ingress is in demo, but Service is in default, Ingress cannot directly reference.  

Handling:  

    1. Place Ingress and Service in the same namespace  
    2. Or redesign routing  
    3. Not recommended to manually forward across namespaces in basic scenarios  

---

## 32. Health Check and ReadinessProbe Issues  

Whether Ingress backend is available fundamentally depends on Service Endpoints.  

Service Endpoints are affected by Pod Ready status.  

Check Pod:  

    kubectl -n demo get pods  

If READY is:  

    0/1  

Check:  

    kubectl -n demo describe pod <pod-name>  

Common probe errors:  

    Readiness probe failed: HTTP probe failed with statuscode: 404  
    connection refused  
    timeout  

Handling:  

    1. Check readinessProbe path  
    2. Check readinessProbe port  
    3. Check application startup time  
    4. Check if health check interface requires authentication  
    5. Check application actual listening address  

---

## 33. Common Issues Quick Reference  

| Status | Common Cause | Priority Check |  
|---|---|---|  
| 404 | Host mismatch | curl with Host |  
| 404 | Path mismatch | Ingress rules paths |  
| 404 | IngressClass mismatch | ingressClassName |  
| 502 | targetPort error | Service port / targetPort |  
| 502 | Backend not listening | PodIP:targetPort |  
| 502 | HTTPS backend forwarded as HTTP | backend-protocol |  
| 503 | Endpoints empty | kubectl get endpoints |  
| 503 | Pod not Ready | kubectl get pods |  
| 413 | Upload size limit | proxy-body-size |  
| 504 | Backend timeout | proxy-read-timeout / backend performance |  
| HTTPS certificate error | Secret or certificate mismatch | TLS Secret |  
| External access failure | DNS / LB / firewall | nslookup / curl NodePort |  

---

## 34. Standard Troubleshooting Command List  

### 34.1 ingress-nginx Status  

    kubectl -n ingress-nginx get pods -o wide  

    kubectl -n ingress-nginx get svc  

    kubectl -n ingress-nginx describe svc ingress-nginx-controller  

    kubectl -n ingress-nginx logs -l app.kubernetes.io/component=controller --tail=200  

---

### 34.2 Ingress Resources  

    kubectl get ingress -A  

    kubectl -n <namespace> get ingress <ingress-name>  

    kubectl -n <namespace> describe ingress <ingress-name>

kubectl -n <namespace> get ingress <ingress-name> -o yaml

---

### 34.3 IngressClass

    kubectl get ingressclass

    kubectl describe ingressclass nginx

    kubectl -n <namespace> get ingress <ingress-name> -o yaml | grep ingressClassName

---

### 34.4 Service and Endpoints

    kubectl -n <namespace> get svc

    kubectl -n <namespace> describe svc <service-name>

    kubectl -n <namespace> get endpoints <service-name>

    kubectl -n <namespace> get endpointslice -l kubernetes.io/service-name=<service-name>

---

### 34.5 Pod Backend

    kubectl -n <namespace> get pods -o wide --show-labels

    kubectl -n <namespace> describe pod <pod-name>

    kubectl -n <namespace> logs <pod-name>

    kubectl -n <namespace> exec -it <pod-name> -- sh

---

### 34.6 Access Test

NodePort HTTP:

    curl -H "Host: demo.ops.local" http://10.0.0.23:30080/

NodePort HTTPS:

    curl -k -H "Host: demo.ops.local" https://10.0.0.23:30443/

LoadBalancer HTTP:

    curl -H "Host: demo.ops.local" http://10.0.0.102/

Direct access Service:

    kubectl run curl-test --image=busybox:1.36 --restart=Never -it --rm -- sh

    wget -qO- http://<service-name>.<namespace>.svc.cluster.local

Direct access PodIP:

    wget -qO- http://<PodIP>:<targetPort>

---

## XXXV. Recommended routing

### 35.1 Visit Return 404

Implementation order:

    kubectl -n ingress-nginx get pods -o wide

    kubectl get ingress -A

    kubectl get ingressclass

    kubectl -n <namespace> describe ingress <ingress-name>

    curl -H "Host: <host>" http://<NodeIP>:<NodePort>/<path>

Focus on identifying:

    1. Host Correct?
    2. Path Correct?
    3. ingressClassName Correct?
    4. Is the request true? ingress-nginx

---

### 35.2 Visit Return 503

Implementation order:

    kubectl -n <namespace> describe ingress <ingress-name>

    kubectl -n <namespace> get svc <service-name>

    kubectl -n <namespace> describe svc <service-name>

    kubectl -n <namespace> get endpoints <service-name>

    kubectl -n <namespace> get pods -o wide --show-labels

Focus on identifying:

    1. Service Existence
    2. Endpoints Is it empty?
    3. Pod Whether or not Ready
    4. selector and label Matches

---

### 35.3 Visit Return 502

Implementation order:

    kubectl -n <namespace> describe svc <service-name>

    kubectl -n <namespace> get endpoints <service-name>

    kubectl -n <namespace> get pods -o wide

    curl / wget Direct access PodIP:targetPort

    kubectl -n ingress-nginx logs -l app.kubernetes.io/component=controller --tail=200

Focus on identifying:

    1. targetPort Correct?
    2. Pod Whether or not to listen to port
    3. Apply listening 0.0.0.0
    4. Backend protocol correct
    5. NetworkPolicy Whether to block

---

## XXXVI. Handling recommendations

Ingress Vulnerable recommendations:

    1. Make sure the request arrives. ingress-nginx
    2. 404 Priority HostI don't know.PathI don't know.IngressClass
    3. 503 Priority Service and Endpoints
    4. 502 Priority targetPort And backend listening.
    5. Don't just look. IngressI'll see. Service and Pod
    6. Don't just look. Pod RunningI'll see. READY
    7. HTTPS We need to check the problem. TLS Secret and certificates SAN
    8. External LB Let's go around. LB Direct measurement NodePort or LoadBalancer IP
    9. Complex annotation It's the easiest thing to do. Ingress Authentication
    10. Changes in the production environment Ingress Prior to confirming impact domain name and path range

---

## XXXVII. Summary

Ingress The access anomaly is essentially a seven-storey entrance link anomaly.

Checking is done from the entrance to the back to confirm layer by layer:

1. Is the DNS resolving to the correct entry  
2. Is NodePort / LoadBalancer reachable  
3. Is ingress-nginx-controller running normally  
4. Is the Ingress recognized by the controller  
5. Does the IngressClass match  
6. Does the Host match  
7. Does the Path match  
8. Does the Service exist  
9. Are Endpoints empty  
10. Are the Pod Ready  
11. Is the targetPort correct  
12. Is the container listening on the port  
13. Is the TLS Secret correct  
14. Are the status codes and upstream in ingress-nginx logs normal  

Experience Judgment:  

1. 404: Likely no rule matched  
2. 503: Likely no available backend  
3. 502: Likely failed to connect to backend  
4. 413: Request body too large  
5. 504: Backend timeout  
6. HTTPS certificate anomaly: Prioritize checking TLS Secret and certificate SAN  

Most Important Commands:  

    kubectl -n <namespace> describe ingress <ingress-name>  

    kubectl -n <namespace> describe svc <service-name>  

    kubectl -n <namespace> get endpoints <service-name>  

    kubectl -n ingress-nginx logs -l app.kubernetes.io/component=controller --tail=200  

Troubleshoot Ingress beyond the Ingress YAML, trace all the way to Service, Endpoints, PodIP, and container listening port.