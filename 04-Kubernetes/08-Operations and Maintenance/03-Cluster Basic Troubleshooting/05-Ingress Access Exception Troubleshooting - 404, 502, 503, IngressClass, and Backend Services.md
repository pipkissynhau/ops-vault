# 05-Ingress Access Exception Troubleshooting: 404, 502, 503, IngressClass, and Backend Services

Recommended Path:

    04-Kubernetes/08-Operations/03-Cluster Basic Troubleshooting/05-Ingress Access Exception Troubleshooting: 404, 502, 503, IngressClass, and Backend Services.md

Tags:

    #Kubernetes
    #Ingress
    #ingress-nginx
    #IngressClass
    #Nginx
    #Service
    #Endpoints
    #HTTP Troubleshooting
    #Cluster Basic Troubleshooting

---

## I. Document Description

This document records the basic troubleshooting methods for Ingress access exceptions in Kubernetes clusters.

Ingress is a commonly used Layer 7 entry resource in Kubernetes, often used in conjunction with components such as ingress-nginx, Traefik, HAProxy Ingress, or cloud provider Ingress Controllers.

This document focuses on using ingress-nginx for troubleshooting.

Common Issues:

    1. Accessing Ingress results in a 404 error
    2. Accessing Ingress results in a 502 error
    3. Accessing Ingress results in a 503 error
    4. Domain name access fails
    5. NodePort is accessible, but Ingress is not
    6. IngressClass does not match
    7. Service backend Endpoints are empty
    8. targetPort configuration is incorrect
    9. ingress-nginx-controller Pod is abnormal
    10. TLS certificate issues
    11. HTTP/HTTPS configurations do not match

Objectives of This Document:

    1. Understand the Ingress access chain
    2. Recognize the common meanings of 404, 502, and 503 errors
    3. Troubleshoot IngressClass issues
    4. Check Host and Path matching
    5. Investigate backend Services
    6. Examine Endpoints
    7. Debug ingress-nginx Controller issues
    8. Analyze ingress-nginx logs
    9. Resolve TLS certificate problems
    10. Establish a standard Ingress troubleshooting process

Applicable Scenarios:

    1. Self-built Kubernetes clusters using kubeadm
    2. Exposing Ingress via NodePort with ingress-nginx
    3. Using ingress-nginx LoadBalancer for exposure
    4. Privately managed Kubernetes clusters
    5. Issues with business domain name access
    6. Abnormalities in Ingress backend services
    7. HTTPS access problems

---

## II. Ingress Access Chain

Typical Ingress access chain:

    User / Client
        |
        v
    Domain name DNS resolution
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
    Ingress rules match Host / Path
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
    Container-based business process

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

Key Points:

    Ingress itself is merely a set of rules.
    The actual traffic reception is handled by the Ingress Controller.
    The backend ultimately relies on Services and Endpoints.

---

## III. Common Status Code Meanings

### 3.1 404 Not Found

Common Meaning:

    The request reached the ingress-nginx, but no matching Ingress rule was found.

Possible Causes:

    1. Host does not match
    2. Path does not match
    3. IngressClassName does not match
    4. Ingress was not recognized by the controller
    5. The request did not include the correct Host header
    6. Default backend was accessed

Typical Error:

    curl http://10.0.0.23:3008010. ingress-nginx Namespace
11. Exposure Method of ingress-nginx
12. Specific Error Status Codes

Example:

Domain name: demo.ops.local
Access entry: 10.0.0.23:30080
Ingress: demo/nginx-demo
IngressClass: nginx
Service: demo/nginx-demo:80
Pod: nginx-demo
Status codes: 404 / 502 / 503

---

## Step Six: Verify the Status of ingress-nginx-controller

Check the Pod:

kubectl -n ingress-nginx get pods -o wide

Expected result:

The ingress-nginx-controller Pod should be Running.

If there are multiple replicas:

All replicas should be in the Running state.
READY should be 1/1 or match the actual number of containers.

Check the Deployment:

kubectl -n ingress-nginx get deploy

View detailed information:

kubectl -n ingress-nginx describe deploy ingress-nginx-controller

If the controller Pod is abnormal, first investigate issues with the controller itself:

ImagePullBackOff
CrashLoopBackOff
Pending
Node NotReady
Configuration errors
admission webhook failures

---

## Step Seven: Verify the Exposure Method of the ingress-nginx Service

Check the Service:

kubectl -n ingress-nginx get svc

Common types:

NodePort
LoadBalancer
ClusterIP

Example for NodePort:

NAME                       TYPE       CLUSTER-IP     PORT(S)
ingress-nginx-controller   NodePort   10.96.10.20    80:30080/TCP,443:30443/TCP

Example for LoadBalancer:

NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP
ingress-nginx-controller   LoadBalancer   10.96.10.20    10.0.0.102

View detailed information:

kubectl -n ingress-nginx describe svc ingress-nginx-controller

Verify the following:

1. Whether the Service exists.
2. Whether the type matches expectations.
3. Whether the HTTP and HTTPS ports are correct.
4. Whether the NodePort is correct.
5. Whether the EXTERNAL-IP has been successfully assigned.
6. Whether there are any Endpoints.

If the ingress-nginx-controller Service does not have Endpoints, it means the controller Pod is not properly connected to the Service backend.

---

## Step Eight: Verify Access to ingress-nginx

### 8.1 Testing via NodePort

Assume the HTTP NodePort is 30080:

curl -I http://10.0.0.23:30080/

If the response is:

HTTP/1.1 404 Not Found

This usually indicates that:

The request has reached ingress-nginx.
However, no matching Ingress rule was found.

This is not a problem; it means that the access path at least leads to the controller.

If the connection fails with errors such as:

Connection refused
Connection timed out
No route to host

Check the following in priority order:

1. Whether the NodePort is correct.
2. Whether the ingress-nginx Service is functioning properly.
3. Whether the ingress-nginx Pod is Running.
4. Whether kube-proxy is working correctly.
5. Whether the node's firewall allows access.
6. Whether security groups or network ACLs are configured properly.

---

### 8.2 Testing via LoadBalancer

Assume the EXTERNAL-IP is 10.0.0.102:

curl -I http://10.0.0.102/

If a 404 error is returned, it means the request reached ingress-nginx.

If access still fails, investigate the following:

1. Whether the MetalLB is functioning correctly.
2. Whether the EXTERNAL-IP has been assigned successfully.
3. Whether the network from the client to the EXTERNAL-IP is accessible.
4. Whether firewalls are blocking access.
5. Whether the ingress-nginx Service is operating normally.

---

## Step Nine: Check Ingress Resources

View all Ingresses:

kubectl get ingress -A

View a specific namespace:

kubectl -n demo get ingress

View detailed information:

kubectl -n demo describe ingress nginx-demo

View the YAML configuration:

kubectl -n demo get ingress nginx-demo -o yaml

Pay special attention to:

1. ingressClassName
2. rules.host
3. paths.path
4. paths.pathType
5. backend.service.name
6. backend.service.port.number/name
7. tls.secretName
8. Events

---

## Step Ten: Check IngressClass

### 10.1 Viewing IngressClass

Execute:

kubectl get ingressclass

Example:

NAME    CONTROLLER             PARAMETERS
nginx   k8s.io/ingress-nginx   <none>

View detailed information:

kubectl describe ingressclass nginx### 12.1 Prefix

Configuration:

    path: /api
    pathType: Prefix

Can match:

    /api
    /api/
    /api/v1

### 12.2 Exact

Configuration:

    path: /api
    pathType: Exact

Can only match:

    /api

Cannot match:

    /api/
    /api/v1

Common issues:

    1. The path is set to /api, but actual access uses /
    2. When pathType is Exact, subpaths do not match
    3. Conflicts occur when multiple Ingress rules have the same path
    4. Missing or incorrect rewrite rules

---

## Chapter Thirteen: Troubleshooting 404 Errors: Ingress Not Recognized by Controller

Check the ingress-nginx logs:

    kubectl -n ingress-nginx logs -l app.kubernetes.io/component=controller --tail=200

You can also check if the controller has loaded the Ingress configuration:

    kubectl -n demo describe ingress nginx-demo

Pay special attention to the Events section.

Common causes:

    1. IngressClass does not match
    2. Incorrect YAML format for the Ingress configuration
    3. The backend Service does not exist
    4. Admission webhook rejection
    5. Abnormal permissions for the ingress-nginx controller

---

## Chapter Fourteen: Troubleshooting 503 Errors: Service Does Not Exist

Check the Ingress backend service:

    kubectl -n demo get ingress nginx-demo -o yaml | grep -A20 backend

Example:

    backend:
      service:
        name: nginx-demo
        port:
          number: 80

Verify the Service exists:

    kubectl -n demo get svc nginx-demo

If it does not exist:

    Error from server (NotFound): services "nginx-demo" not found

Steps to resolve:

    1. Create a Service if it does not exist.
    2. Correct the service name in the Ingress configuration.
    3. Ensure the namespace is correct.

Note:

    The Service referenced by the Ingress must be within the same namespace.
    Ordinary Ingress backends do not reference Services across different namespaces.

---

## Chapter Fifteen: Troubleshooting 503 Errors: No Endpoints Available

Check the Service configuration:

    kubectl -n demo describe svc nginx-demo

Focus on the Endpoints field:

    Endpoints: <none>

If this is the case, it means the Service has no available backend.

Continue to check:

    kubectl -n demo get endpoints nginx-demo
    kubectl -n demo get endpointslice -l kubernetes.io/service-name=nginx-demo

Common reasons for no Endpoints:

    1. The Service selector does not match the Pod label.
    2. The Pod is not in the Ready state.
    3. ReadinessProbe failures.
    4. The Pod is in a different namespace.
    5. The Service has no selector defined.
    6. The Pod has been restarted or is experiencing issues.

---

### 15.1 Checking Selector and Label

Verify the Service selector:

    kubectl -n demo get svc nginx-demo -o yaml | grep -A10 selector

Check the Pod label:

    kubectl -n demo get pods --show-labels

Search for Pods using the selector:

    kubectl -n demo get pods -l app=nginx-demo -o wide

If no Pods are found, it indicates that the selector does not match.

---

### 15.2 Checking Pod Readiness

View the Pods status:

    kubectl -n demo get pods -o wide

Pay attention to the READY status:

    0/1
    1/1

If it shows 0/1, check the specific Pod with:

    kubectl -n demo describe pod <pod-name>

Check for errors such as:

    Readiness probe failed
    connection refused
    HTTP probe failed
    timeout

Steps to resolve:

    1. Adjust the readinessProbe settings.
    2. Verify the actual health check path used by the application.
    3. Ensure the probe port is correct.
    4. Increase the initialDelaySeconds if necessary.
    5. Fix any issues with the application itself.

---

## Chapter Sixteen: Troubleshooting 502 Errors: Incorrect targetPort

Check the Service configuration:

    kubectl -n demo get svc nginx-demo -o yaml

Example:

    ports:
    - port: 80
      targetPort: 8080

Explanation:

    Ingress -> Service:80 -> PodIP:8080

If the container actually listens on port 80 but the targetPort is set to 8080, it will result in a 502### Common Log Issues:

- **No endpoints available**: The Service backend is empty.
- **Service not found**: The Service referenced by the Ingress does not exist.
- **Connect() failed**: Failed to connect to the backend, often due to an incorrect targetPort or the application not being listening.
- **Upstream timed out**: The backend response timed out, possibly due to slow business processing, network interruptions, or insufficient timeout settings.

---

## 21. Viewing Ingress-nginx Dynamic Configuration (Optional)

Ingress-nginx generates Nginx configurations based on the Ingress settings.

To access the controller Pod:

```
kubectl -n ingress-nginx exec -it <controller-pod-name> -- sh
```

To view the Nginx configuration:

```
nginx -T
```

Or filter by domain name:

```
nginx -T | grep -n "demo.ops.local" -A30
```

**Note**: This operation is suitable for advanced troubleshooting. For basic issues, check the Ingress, Service, Endpoints, and logs first.

---

## 22. HTTPS/TLS Troubleshooting

### 22.1 Viewing Ingress TLS Configuration

Execute:

```
kubectl -n demo get ingress nginx-demo -o yaml | grep -A10 tls
```

**Example**:

```
tls:
- hosts:
  - demo.ops.local
  secretName: demo-tls
```

**Verification**:

1. Verify that `tls.hosts` is correct.
2. Confirm the `secretName` is accurate.
3. Ensure the Secret exists.
4. Check if the Secret type is `kubernetes.io/tls`.

---

### 22.2 Checking the TLS Secret

View the Secret:

```
kubectl -n demo get secret demo-tls
```

Check the type:

```
kubectl -n demo get secret demo-tls -o yaml | grep type
```

**Expected value**: `type: kubernetes.io/tls`

Check the fields:

```
kubectl -n demo get secret demo-tls -o jsonpath '{.data}' | head
```

It should contain:

- `tls.crt`
- `tls.key`

---

### 22.3 Viewing Certificate Content

Check the validity period:

```
kubectl -n demo get secret demo-tls \
  -o jsonpath '{.data.tls\.crt}' | base64 -d | openssl x509 -noout -dates
```

View the SAN:

```
kubectl -n demo get secret demo-tls \
  -o jsonpath '{.data.tls\.crt}' | base64 -d | openssl x509 -noout -text | grep -A2 "Subject Alternative Name"
```

Confirm that the domain name is listed in the certificate's SAN.

---

### 22.4 Testing HTTPS with curl

**Self-signed certificate test**:

```
curl -k -H "Host: demo.ops.local" https://10.0.0.23:30443/
```

**Official certificate test**:

```
curl -H "Host: demo.ops.local" https://10.0.0.23:30443/
```

If the certificate is not trusted, use `-k` for temporary verification, but do not rely on it in production.

---

### 22.5 Checking cert-manager-related Settings

If the certificates are managed by cert-manager:

```
kubectl -n demo get certificate
kubectl -n demo describe certificate <certificate-name>
kubectl -n cert-manager get pods -o wide
kubectl -n cert-manager logs deploy/cert-manager --tail=100
```

**Common issues**:

1. The Certificate is not Ready.
2. The ClusterIssuer does not exist.
3. DNS-01/HTTP-01 verification fails.
4. The Secret has not been generated.
5. The Secret name does not match the Ingress configuration.

---

## 23. Ingress Annotation Issues

Many advanced features of ingress-nginx are controlled through annotations.

View Ingress annotations:

```
kubectl -n demo get ingress nginx-demo -o yaml | grep -A30 annotations
```

**Common annotations**:

- `nginx.ingress.kubernetes.io/rewrite-target`
- `nginx.ingress.kubernetes.io/backend-protocol`
- `nginx.ingress.kubernetes.io/proxy-body-size`
- `nginx.ingress.kubernetes.io/proxy-read-timeout`
- `nginx.ingress.kubernetes.io/proxy-send-timeout`
- `nginx.ingress.kubernetes.io/ssl-redirect`
- `nginx.ingress.kubernetes.io/use-regex`

**Common issues**:

1. Incorrect `rewrite-target` configuration may cause path forwarding errors.
2. Missing `backend-protocol` may result in a 502 errorIf there are additional components in front of Ingress, such as:

    F5
    Nginx
    HAProxy
    SLB
    ELB
    Firewall
    WAF

It is necessary to confirm the following:

    1. Whether the external LoadBalancer is listening on ports 80/443.
    2. Whether the external LoadBalancer is forwarding requests to the correct NodePort.
    3. Whether the health checks are functioning properly.
    4. Whether the Host header is being preserved in the requests.
    5. Whether HTTPS termination is being performed correctly.
    6. Whether there are any duplicate TLS terminations.
    7. Whether the path is being modified during the forwarding process.
    8. Whether any request headers are being intercepted.

Methods for troubleshooting:

    1. Try accessing the NodePort directly, bypassing the external LoadBalancer.
    2. Directly use `curl` to access the IP address of the ingress-nginx LoadBalancer.
    3. Compare the results of external and internal accesses.
    4. Check the logs of the external LoadBalancer.

---

## 29. Impact of NetworkPolicy

If NetworkPolicy is enabled, the traffic from the ingress-nginx-controller to the business Pods may be blocked.

To check:

    Use `kubectl get networkpolicy -A` to view all NetworkPolicies.
    Verify the backend namespace by using `kubectl -n demo get networkpolicy`.

Permissions required:

    The controller Pod in the ingress-nginx namespace must have access to the business ports of the Pods in the demo namespace.

Common issues:

    1. The Service Endpoints are not empty, but access from ingress-nginx times out.
    2. The PodIP and port exist, but there is still a timeout issue.
    3. Error messages like "upstream timed out" appear in the logs.

Production tips:

    Do not delete NetworkPolicies arbitrarily. Make sure to allow only necessary traffic based on the business access requirements.

---

## 30. Issues with Service and Ingress Port References

The Ingress backend can reference the port number or name of a Service.

Example for Service configuration:

    ```yaml
    ports:
      - name: http
        port: 80
        targetPort: 8080
    ```
    Example for Ingress configuration:

    ```yaml
    service:
      name: nginx-demo
      port:
        number: 80
    ```

Or:

    ```yaml
    service:
      name: nginx-demo
      port:
        name: http
    ```

Common errors:

    1. The Ingress specifies a targetPort of 8080, but the Service’s actual port is 80.
    2. The Ingress uses the name "web", but the Service’s port name is "http".
    3. The Service’s targetPort is specified incorrectly.
    4. Wrong ports are referenced when using a multi-port Service.

To resolve these issues:

    Use `kubectl -n demo get svc nginx-demo -o yaml` to view the Service configuration.
    Then use `kubectl -n demo get ingress nginx-demo -o yaml` to check the Ingress configuration.

---

## 31. Issues with Ingress and Service Namespaces

The Service that the Ingress backend references must be in the same namespace.

Example:

    If the Ingress is in the `demo` namespace, the Service must also be in the `demo` namespace.

To verify:

    Use `kubectl get ingress -A` and `kubectl get svc -A`.

If the Ingress and Service are in different namespaces, the Ingress will not be able to reference the Service directly.

Solutions:

    1. Move the Ingress and Service to the same namespace.
    2. Re-design the routing mechanism if necessary.
    3. In basic scenarios, it is not recommended to manually forward requests across different namespaces.

---

## 32. Issues with Health Checks and ReadinessProbes

The availability of the Ingress backend essentially depends on the Service Endpoints.

Service Endpoints are in turn affected by the Pod’s Ready status.

To check Pods:

    Use `kubectl -n demo get pods`.

If a Pod is not ready, its status will be `0/1`.

To view detailed information about a Pod:

    Use `kubectl -n demo describe pod <pod-name>`.

Common probe errors:

    `Readiness probe failed: HTTP probe failed with statuscode: 404`
    `connection refused`
    `timeout`

To resolve these issues:

    1. Verify the path specified in the readinessProbe.
    2. Check the port number used by the readinessProbe.
    3. Ensure that the application has started successfully.
    4. Verify whether authentication is required for the health check API.
    5.Direct access to PodIP:

    wget -qO- http://<PodIP>:<targetPort>

---

## Section 35: Recommended Troubleshooting Paths

### 35.1 Access returns 404

Sequence of actions:

    kubectl -n ingress-nginx get pods -o wide

    kubectl get ingress -A

    kubectl get ingressclass

    kubectl -n <namespace> describe ingress <ingress-name>

    curl -H "Host: <host>" http://<NodeIP>:<NodePort>/<path>

Key points to confirm:

    1. Whether the Host is correct.
    2. Whether the Path is correct.
    3. Whether the ingressClassName is correct.
    4. Whether the request actually reaches ingress-nginx.

---

### 35.2 Access returns 503

Sequence of actions:

    kubectl -n <namespace> describe ingress <ingress-name>

    kubectl -n <namespace> get svc <service-name>

    kubectl -n <namespace> describe svc <service-name>

    kubectl -n <namespace> get endpoints <service-name>

    kubectl -n <namespace> get pods -o wide --show-labels

Key points to confirm:

    1. Whether the Service exists.
    2. Whether the Endpoints are empty.
    3. Whether the Pod is Ready.
    4. Whether the selector and label match.

---

### 35.3 Access returns 502

Sequence of actions:

    kubectl -n <namespace> describe svc <service-name>

    kubectl -n <namespace> get endpoints <service-name>

    kubectl -n <namespace> get pods -o wide

    curl / wget direct access to PodIP:targetPort

    kubectl -n ingress-nginx logs -l app.kubernetes.io/component=controller --tail=200

Key points to confirm:

    1. Whether the targetPort is correct.
    2. Whether the Pod is listening on that port.
    3. Whether the application is listening on 0.0.0.0.
    4. Whether the backend protocol is correct.
    5. Whether NetworkPolicy is blocking access.

---

## Section 36: Handling Suggestions

Suggestions for troubleshooting Ingress issues:

    1. First, confirm whether the request reaches ingress-nginx.
    2. For 404 errors, check Host, Path, and IngressClass first.
    3. For 503 errors, check Service and Endpoints first.
    4. For 502 errors, check targetPort and backend listening settings.
    5. Don't focus only on Ingress; also check Service and Pod status.
    6. Ensure the Pod is not just Running but also Ready.
    7. For HTTPS issues, verify TLS Secret and certificate SAN.
    8. If there is an external LB, bypass it and directly test NodePort or LoadBalancer IP.
    9. When encountering complex annotation issues, simplify the Ingress configuration for verification.
    10. Before modifying Ingress in a production environment, confirm the impact on domain names and path ranges.

---

## Section 37: Summary

Ingress access anomalies are essentially issues with the seven-layer entry link. During troubleshooting, check each layer from the entrance to the backend:

    1. Whether DNS resolves to the correct entry point.
    2. Whether NodePort / LoadBalancer is accessible.
    3. Whether ingress-nginx-controller is functioning normally.
    4. Whether Ingress is recognized by the controller.
    5. Whether IngressClass matches the configuration.
    6. Whether the Host matches the request.
    7. Whether the Path matches the request.
    8. Whether the Service exists.
    9. Whether Endpoints are available.
    10. Whether the Pod is Ready.
    11. Whether the targetPort is correct.
    12. Whether the container is listening on that port.
    13. Whether the TLS Secret is valid.
    14. Check the status codes and upstream responses in ingress-nginx logs.

Judgment based on experience:

    1. 404: Usually indicates that no rule matches the request.
    2. 503: Often means there is no available backend service.
    3. 502: Typically indicates a failure to connect to the backend.
    4. 413: The request body is too large.
    5. 504: The backend timed out.
    6. HTTPS certificate issues: Check TLS Secret and certificate SAN first.

Most important commands:

    kubectl -n <namespace> describe ingress <ingress-name>

    kubectl -n <namespace> describe svc