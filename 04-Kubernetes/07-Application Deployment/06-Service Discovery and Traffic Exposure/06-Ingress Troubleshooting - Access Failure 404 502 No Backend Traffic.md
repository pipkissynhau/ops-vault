# 06-Ingress Common Troubleshooting: Access Issues, 404, 502, and No Backend Traffic

## Document Notes
- Document Position: In the Kubernetes service discovery and traffic exposure phase, the small stage summary of Ingress learning
- Applicable Stage: After understanding the role of Ingress, the basic principles of Ingress Controller, and completing the first minimal practical implementation based on kubeadm + ingress-nginx + NodePort, used to establish an entry fault diagnosis approach
- Recommended Path: `04-Kubernetes/07-Apply deployment/06-Service detection and traffic exposure/06-Ingress Question Question Question Question Question Question Question Question404I don't know.502 No traffic with backend.md`

## Tags
#Kubernetes #Ingress #IngressController #ingress-nginx #Service #Endpoints #Pod #HTTP #404 #502 #TheBarrier. #kubeadm #NodePort #ApplyDeployment #Clouds. #Transport #InterviewNotes

---

## I. Article Objectives

This article organizes the minimal troubleshooting path for common ingress-nginx entry faults based on the current actual environment.

Current environment prerequisites are as follows:

- Nodes:
  - `10.0.0.20`
  - `10.0.0.21`
  - `10.0.0.22`
- ingress-nginx Controller:
  - HTTP NodePort: `31152`
  - HTTPS NodePort: `30361`
- IngressClass:
  - `nginx -> k8s.io/ingress-nginx`

After completing this article, the following capabilities should be formed:

- Ability to trace Ingress issues along the real request chain
- Ability to distinguish between "entry issues", "rule issues", and "backend issues"
- Ability to understand common causes of 404, 502, 503, and no backend traffic
- Ability to master the most commonly used verification commands

---

## II. General Principles of Ingress Troubleshooting

Ingress troubleshooting should not focus only on a single YAML file, but rather trace the real traffic path layer by layer.

The typical chain in the current environment is as follows:

    Client
    -> Domain/hosts
    -> 10.0.0.20:31152 / 10.0.0.21:31152 / 10.0.0.22:31152
    -> ingress-nginx Controller
    -> Ingress rule matching
    -> Service
    -> Endpoints
    -> Pod
    -> Container process

Any issue in any layer of this chain may manifest as entry access anomalies.

---

## III. Fixed Troubleshooting Order

It is recommended to troubleshoot in the following fixed order:

### Layer 1: Confirm the request reaches the correct entry
Verify:

- Whether the hosts or domain name resolves to the correct node IP
- Whether the node IP is reachable
- Whether the current access port is `31152` or `30361`

### Layer 2: Confirm Controller is normal
Verify:

- Whether the ingress-nginx Controller Pod is `Running`
- Whether `ingress-nginx-controller` Service exists
- Whether the Service type is `NodePort`

### Layer 3: Confirm Ingress is being managed and matched
Verify:

- Whether `ingressClassName` is correct
- Whether the host is correct
- Whether the path is correct
- Whether the Host header in the request matches the rule

### Layer 4: Confirm Service is correct
Verify:

- Whether the backend Service name is correct
- Whether the backend Service port is correct

### Layer 5: Confirm Endpoints are valid
Verify:

- Whether the Service selects the Pod
- Whether the Pod is Ready
- Whether Endpoints is empty

### Layer 6: Confirm the application is normal
Verify:

- Whether the Pod is running normally
- Whether the container process listens on the correct port
- Whether the application itself can return normal responses

---

## IV. First Category of Issues: Complete Access Failure

### Common Phenomena

For example:

- Browser cannot open
- Curl timeout
- Connection refused
- No HTTP response

### Troubleshooting Order

#### 1. First confirm Controller is normal

    kubectl get pods -n ingress-nginx
    kubectl get svc -n ingress-nginx

Focus on confirming:

- `ingress-nginx-controller-xxxxx` is `Running`
- `ingress-nginx-controller` is `NodePort`
- HTTP NodePort is `31152`
- HTTPS NodePort is `30361`

#### 2. Then confirm the entry address is correct

In the current environment, you can directly test:

    curl http://10.0.0.20:31152
    curl http://10.0.0.21:31152
    curl http://10.0.0.22:31152

If even this step fails, prioritize troubleshooting:

- Whether the local machine can reach the node IP
- Node host firewall
- Cloud host security group
- Whether NodePort is blocked

#### 3. Then confirm Ingress exists

    kubectl get ingress -A
    kubectl describe ingress <ingress-name> -n <namespace>

Confirm:

- Ingress object has been created
- host, path, backend are correct
- No obvious event errors

---

## V. Second Category of Issues: Returns 404

### Common Phenomena

For example:

- The entry can be accessed
- But returns 404 quickly

This usually indicates:

> The request has reached the ingress-nginx Controller, but no matching rule was found.

### Most Common Causes

#### 1. Host mismatch

For example, the Ingress writes:

    host: nginx.example.com

But actual access is:

- Using an incorrect domain name
- Direct IP access
- Curl without Host header
- Hosts configuration error

#### 2. Path mismatch

For example, the rule matches:

    /api

But actual access is:

    /

#### 3. ingressClassName mismatch

If the Ingress is not being managed by ingress-nginx, it may also manifest as a default 404.

---

## VI. 404 Troubleshooting Methods

### Step 1: Validate with Host Header

In the current environment, it is recommended to directly execute:

    curl -H "Host: nginx.example.com" http://10.0.0.20:31152

If needed, you can also replace with:

    curl -H "Host: nginx.example.com" http://10.0.0.21:31152
    curl -H "Host: nginx.example.com" http://10.0.0.22:31152

### Step 2: Check Ingress Rules

kubectl describe ingress <ingress-name> -n <namespace>

Key Verifications:

- `CLASS` is `nginx`
- `HOSTS` is `nginx.example.com`
- path matches current request
- backend is correct

### Step 3: Check IngressClass

    kubectl get ingressclass
    kubectl get ingress -n <namespace> -o yaml

Confirm:

- `ingressClassName: nginx`
- Cluster has `IngressClass/nginx`

---

## VII. Third Type of Issue: Returns 502 or 503

### Common Symptoms

For example:

- Request reached ingress-nginx Controller
- Rules appear to match
- But returns 502 Bad Gateway or 503 Service Unavailable

This typically indicates:

> The ingress layer is likely established, the issue is more likely in the backend Service / Endpoints / Pod.

### Most Common Causes

#### 1. Backend Service name is incorrect
#### 2. Backend Service port is incorrect
#### 3. Service exists but Endpoints are empty
#### 4. Pod is not Ready
#### 5. Container is not listening on correct port

---

## VIII. Troubleshooting 502/503 Issues

### Step 1: Check Ingress backend

    kubectl describe ingress <ingress-name> -n <namespace>

Key Verifications:

- Backend Service name
- Backend port

### Step 2: Check Service

    kubectl get svc -n <namespace>

Confirm:

- Service name is correct
- Exposed port is correct

### Step 3: Check Endpoints

    kubectl get endpoints -n <namespace>
    kubectl describe svc <service-name> -n <namespace>

If Endpoints are empty, the issue is likely:

- selector
- Pod labels
- Pod Ready

### Step 4: Check Pod

    kubectl get pods -n <namespace> -o wide
    kubectl describe pod <pod-name> -n <namespace>
    kubectl logs <pod-name> -n <namespace>

Confirm:

- Pod is `Running`
- Ready status is normal
- No obvious container errors
- Application is listening on correct port

### Step 5: Validate Service from within cluster

    kubectl run test-shell --rm -it --image=busybox:1.36 -n <namespace> -- sh

After entering, execute:

    wget -qO- http://<service-name>:<port>

If Service access fails within cluster, the issue is not with Ingress but the backend business chain.

---

## IX. Fourth Type of Issue: Ingress Created but No Backend Traffic

### Common Symptoms

For example:

- Ingress exists
- Pod and Service are normal
- But business logs show no requests
- Backend appears to receive no traffic

This typically indicates:

> Requests may not have matched the current business Ingress rules.

### Common Causes

- Request didn't reach Controller
- Host mismatch
- Path mismatch
- Influenced by other Ingress rules

---

## X. Troubleshooting When No Backend Traffic

### 1. Check Controller Logs

    kubectl logs -n ingress-nginx <controller-pod-name>

If no request records, prioritize checking:

- hosts
- node IP
- NodePort
- network connectivity from local to node

### 2. Check Host Correctness

In current environment, always use:

    curl -H "Host: nginx.example.com" http://10.0.0.20:31152

This forces requests to hit the target Host rule.

### 3. Check Path Matching

If Ingress only handles:

    /api

But actual access is:

    /

Then the rule won't match.

### 4. Check for Other Ingress Rules

    kubectl get ingress -A

When multiple Ingresses exist, view the full rule set rather than individual YAMLs.

---

## XI. A Practical Habit: Forced Layer-by-Layer Validation

When issues are unclear, perform forced layer-by-layer validation.

### Layer 1: Validate Application Itself
Confirm Pod, container, and process are normal.

### Layer 2: Validate Service
Directly access from cluster:

    http://service-name:port

### Layer 3: Validate Controller Entry
Directly test:

    curl http://10.0.0.20:31152
    curl http://10.0.0.21:31152
    curl http://10.0.0.22:31152

### Layer 4: Validate Ingress Rules
Test with Host header:

    curl -H "Host: nginx.example.com" http://10.0.0.20:31152

### Layer 5: Validate hosts or domain
Finally validate:

    curl http://nginx.example.com:31152

This approach helps quickly narrow down the issue scope.

---

## XII. Why `kubectl get all` Can't See Ingress

Common phenomenon at this stage:

    kubectl get all -n test

Cannot see:

- Ingress

This is normal behavior.  
Reason:

> `kubectl get all` by default does not display Ingress.

To view simultaneously, execute:

    kubectl get ingress -n test

Or:

    kubectl get pod,svc,deploy,rs,ing -n test

---

## XIII. Most Common Command Set for Troubleshooting

### View Controller

    kubectl get pods -n ingress-nginx
    kubectl get svc -n ingress-nginx
    kubectl logs -n ingress-nginx <controller-pod-name>

### View Ingress /think

kubectl get ingress -A  
kubectl get ingress -n <namespace> -o wide  
kubectl describe ingress <ingress-name> -n <namespace>  
kubectl get ingress -n <namespace> -o yaml  

### View IngressClass  

    kubectl get ingressclass  

### View Backend Service and Endpoints  

    kubectl get svc -n <namespace>  
    kubectl get endpoints -n <namespace>  
    kubectl describe svc <service-name> -n <namespace>  

### View Backend Pod  

    kubectl get pods -n <namespace> -o wide  
    kubectl describe pod <pod-name> -n <namespace>  
    kubectl logs <pod-name> -n <namespace>  

### Manual Request Validation  

    curl http://10.0.0.20:31152  
    curl -H "Host: nginx.example.com" http://10.0.0.20:31152  

---

## Fourteen. Interview Answer Reference  

If asked "How to troubleshoot Ingress issues," I would respond as follows:  

In a kubeadm + ingress-nginx + NodePort environment, I would first confirm whether the request reaches the correct node IP and NodePort, such as the current environment's `31152`.  
Then I would check whether the ingress-nginx Controller Pod and Service are normal, followed by verifying whether the Ingress is properly managed, including `ingressClassName`, host, path, and backend Service.  
Next, I would continue checking the backend Service and Endpoints to confirm whether the Service selects the correct Pod and whether Endpoints are empty.  
Finally, I would check whether the Pod and application listening ports are normal.  
To quickly validate whether the rule is matched, I would prioritize executing:  

    curl -H "Host: nginx.example.com" http://<节点IP>:31152  

This helps quickly determine whether the issue lies in the ingress layer, rule layer, or backend business layer.  

---

## Fifteen. Stage Conclusion  

After completing this article, the following conclusions should be formed:  

- Ingress troubleshooting should follow the real request chain for layer-by-layer investigation  
- Current environment priorities should focus on:  
  - Node IP  
  - HTTP NodePort: `31152`  
  - ingress-nginx Controller  
  - Ingress rules  
  - Service / Endpoints  
  - Pod / Application  
- 404 often indicates rule mismatch  
- 502 / 503 often indicate backend chain issues  
- `curl -H "Host: ..."` is the most critical first-round verification command  
- `kubectl get all` defaults to not showing Ingress, should be viewed separately  

---

## Keyword Mnemonics  

- Complete failure: Prioritize checking node IP, NodePort, Controller  
- 404: Prioritize checking Host, Path, ingressClassName  
- 502 / 503: Prioritize checking Service, Endpoints, Pod  
- `curl -H "Host: ..."`: Key method for verifying rule matching  
- `kubectl get all`: Defaults to not showing Ingress  

---

## Next Day Suggestions  

This phase has been completed. Suggest returning to the mainline content of application deployment for subsequent sections [[01-PVC and StorageClass Foundation: Sustainable storage entry ]].  

Later, when needed, expand on:  

- TLS / HTTPS basics  
- Relationship between ELB / ALB and Ingress in cloud-managed clusters  
- More complex Ingress annotations and advanced features