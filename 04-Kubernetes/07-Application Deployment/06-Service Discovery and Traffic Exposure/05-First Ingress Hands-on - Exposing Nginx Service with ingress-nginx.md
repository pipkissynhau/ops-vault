# 05-First Ingress Practical: Exposing Nginx Service Using ingress-nginx

## Document Notes
- Document Positioning: In the Kubernetes service discovery and traffic exposure phase, the first complete implementation practice for learning Ingress
- Applicable Stage: After understanding the role of Ingress, the basic principles of Ingress Controller, and completing the installation of ingress-nginx Controller and NodePort entry preparation in a kubeadm cluster, start to establish an entry chain through a minimal closed-loop example
- Recommended Path: `04-Kubernetes/07-Apply deployment/06-Service detection and traffic exposure/05-First Ingress Operational: based on ingress-nginx Exposure Nginx Services.md`

## Tags
#Kubernetes #Ingress #IngressController #ingress-nginx #Nginx #Service #NodePort #HTTP #DomainName #PathForward #kubeadm #ApplyDeployment #Clouds. #Transport #InterviewNotes

---

## One. Article Objectives

This article completes a minimal Ingress closed-loop experiment based on the actual environment in the current kubeadm cluster, exposing the Nginx service through ingress-nginx.

After completing this article, the following objectives should be achieved:

- Can deploy a minimal Nginx Deployment
- Can create a ClusterIP Service as an Ingress backend
- Can create an Ingress taken over by ingress-nginx
- Can understand the complete request flow from `NodesIP:NodePort` to the Controller, then forwarded to the backend Service and Pod
- Can use `curl -H "Host: ..."` to verify if Ingress matches
- Can thoroughly understand the meaning of each key field in the Ingress YAML

---

## Two. Current Environment Information

Current cluster nodes are as follows:

- `k8s-master`:`10.0.0.20`
- `k8s-node01`:`10.0.0.21`
- `k8s-node02`:`10.0.0.22`

Current `ingress-nginx-controller` entry is as follows:

- HTTP:`31152`
- HTTPS:`30361`

Current `IngressClass` is as follows:

- `nginx -> k8s.io/ingress-nginx`

Therefore, the current entry should be understood as:

### HTTP Entry
    http://10.0.0.20:31152
    http://10.0.0.21:31152
    http://10.0.0.22:31152

### HTTPS Entry
    https://10.0.0.20:30361
    https://10.0.0.21:30361
    https://10.0.0.22:30361

---

## Three. Experiment Objectives

The objectives of this experiment are as follows:

- Deploy a business Deployment named `nginx-web`
- Create a ClusterIP Service named `nginx-web-svc`
- Create an Ingress named `nginx-web-ingress`
- Make requests to `nginx.example.com` forwarded through the ingress-nginx Controller to `nginx-web-svc:80`

Expected access method is as follows:

    http://nginx.example.com:31152

Or verify through the Host header without modifying hosts:

    curl -H "Host: nginx.example.com" http://10.0.0.20:31152

---

## Four. Prerequisites

Before creating the business Ingress, the following conditions should be met:

### 1. ingress-nginx Controller is installed and running normally

Execute:

    kubectl get pods -n ingress-nginx
    kubectl get svc -n ingress-nginx
    kubectl get ingressclass

Should at least confirm:

- `ingress-nginx-controller-xxxxx` is `Running`
- `ingress-nginx-controller` is `NodePort`
- HTTP NodePort is `31152`
- HTTPS NodePort is `30361`
- `IngressClass/nginx` exists

### 2. Current environment entry is not the business Service, but the Controller NodePort

That is:

- Client first accesses `NodesIP:31152`
- Then the ingress-nginx Controller forwards to the backend business Service according to Ingress rules

---

## Five. Deploy Backend Business: Nginx Deployment

### Deployment YAML

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-web
      namespace: test
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: nginx-web
      template:
        metadata:
          labels:
            app: nginx-web
        spec:
          containers:
            - name: nginx
              image: nginx:1.27
              ports:
                - containerPort: 80

### Notes

#### 1. `name: nginx-web`
Indicates the Deployment name is:

    nginx-web

#### 2. `namespace: test`
Indicates the business resource is created in:

    test

namespace. The subsequent Service and Ingress should also be in the same namespace.

#### 3. `replicas: 2`
Indicates two Nginx Pods are created, forming a minimal backend replica group.

#### 4. `matchLabels` and `template.labels`
Use the same label:

    app: nginx-web

The Service will select backend Pods through this label later.

#### 5. `containerPort: 80`
Indicates the Nginx container listens on port 80.

### Apply Command

    kubectl create namespace test
    kubectl apply -f nginx-web-deployment.yaml

### Check Command

    kubectl get pods -n test -o wide

### Expected Results

Should see two Pods with status similar to:

- `Running`
- `READY 1/1`

---

## Six. Create Backend Service

### Service YAML

apiVersion: v1
kind: Service
metadata:
  name: nginx-web-svc
  namespace: test
spec:
  selector:
    app: nginx-web
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP

### Explanation

This Service's responsibilities are:

- Provide a stable access entry point for backend Pods
- Serve as the backend target for subsequent Ingress

In the current stage, it does not directly assume the responsibility of an external cluster entry point, so the type should be:

    ClusterIP

### Field Understanding

#### 1. `name: nginx-web-svc`
This is the name of the Service that the subsequent Ingress backend will reference.

#### 2. `selector: app: nginx-web`
Indicates that this Service selects backend Pods through labels.

#### 3. `port: 80`
Indicates that the Service itself provides a service port of 80 to the outside.

#### 4. `targetPort: 80`
Indicates that after receiving traffic, the Service forwards it to the Pod's port 80.

#### 5. `type: ClusterIP`
Indicates that this Service only provides access within the cluster and does not serve as an external cluster entry point.

### Apply Command

    kubectl apply -f nginx-web-svc.yaml

### Check Command

    kubectl get svc -n test
    kubectl get endpoints -n test

### Expected Results

Confirm:

- `nginx-web-svc` has been successfully created
- `Endpoints` is not empty

If `Endpoints` is empty, it indicates that the Service has not correctly selected the Pod. Even if the Ingress is created successfully later, traffic will not reach the Pod.

---

## Seven. Creating Ingress

### Ingress YAML

    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: nginx-web-ingress
      namespace: test
    spec:
      ingressClassName: nginx
      rules:
        - host: nginx.example.com
          http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: nginx-web-svc
                    port:
                      number: 80

### Overall Meaning of This YAML

The overall function of this Ingress can be understood as:

- When requests enter the ingress-nginx Controller
- And the request's Host is `nginx.example.com`
- And the path matches `/`
- The traffic is forwarded to `test` namespace's `nginx-web-svc:80`

It is essentially not "creating an entry program," but rather "declaring an entry forwarding rule."

In other words:

- The ingress-nginx Controller is responsible for receiving requests
- The Ingress YAML tells the Controller: "Where should this request be forwarded?"

---

## Eight. Detailed Field-by-Field Explanation of Ingress YAML

This section is recommended to focus on understanding, as this is the first time writing an Ingress rule as a configuration.

### 1. `apiVersion: networking.k8s.io/v1`

Indicates the API group and version to which this resource belongs.

The meaning here is:

- This is a resource in the `networking.k8s.io` API group in Kubernetes
- Using the `v1` version

At this stage, you can first understand it as:

> Kubernetes uses this as a version declaration to identify "this is an Ingress resource configuration."

---

### 2. `kind: Ingress`

Indicates the type of resource created by this YAML:

- `Ingress`

This line determines how Kubernetes will process this configuration. Writing `Ingress` means:

> This is not a workload object, not a service object, but a configuration for an entry rule.

---

### 3. `metadata`

`metadata` is used to describe the basic information of this object.

In this YAML, the most important parts are:

- `name`
- `namespace`

---

### 4. `metadata.name: nginx-web-ingress`

Indicates that the name of this Ingress rule object is:

    nginx-web-ingress

Its main purpose is:

- To facilitate identification and management of this rule
- For subsequent lookup of object details, for example:

    kubectl describe ingress nginx-web-ingress -n test

This name is not for client access, but for Kubernetes management objects.

---

### 5. `metadata.namespace: test`

Indicates that this Ingress resource is created in:

    test

namespace.

This is very important because the Service referenced by the Ingress backend is usually also located in the same namespace.

In other words:

- This Ingress is in `test`
- The referenced `nginx-web-svc` is also in `test`

If the namespaces are inconsistent, it may result in:

- The Ingress object is created successfully
- But the backend Service cannot be found

Therefore, it's important to establish a habit at this stage:

> Understand that Ingress, backend Service, and backend business Pod should initially be placed in the same namespace.

---

### 6. `spec`

`spec` represents the core part of this Ingress, indicating:

> The specific content of this Ingress rule.

You can understand it as:

- `metadata` is "what is the name and location of this rule"
- `spec` is "how this rule forwards traffic"

The most critical fields are located under `spec`.

---

### 7. `spec.ingressClassName: nginx`

This line indicates: /think

> This Ingress rule is handled by the Controller associated with the IngressClass named `nginx`.

Current environment:

- The name of `IngressClass` is `nginx`
- Its corresponding Controller is `k8s.io/ingress-nginx`

So writing:

    ingressClassName: nginx

means:

- This rule should be taken over by the currently installed ingress-nginx

This line is very critical.  
If written incorrectly, for example:

- Using a non-existent class name
- Mismatching with the IngressClass in the current cluster

It may result in:

- Ingress object creation succeeds
- But ingress-nginx does not take over
- Final access fails

At this stage, just remember:

> `ingressClassName` determines "who is responsible for this rule".

---

### 8. `spec.rules`

`rules` means:

> A set of entry rules defined in this Ingress.

It's a list because theoretically, one Ingress can define multiple rule groups, for example:

- Multiple hosts
- Multiple paths
- Multiple backends

In this minimal example, only one rule group is defined, so only one `-` entry is seen.

You can understand `rules` as:

> The collection of routing rules for Ingress.

---

### 9. `- host: nginx.example.com`

This line means:

> This rule only takes effect for requests with Host set to `nginx.example.com`.

This is a critical layer in Ingress's seven-layer rules:

- Not just checking IP and port
- Also checks the Host in HTTP request headers

In the current environment, this means if a request enters the Controller:

- If the Host is `nginx.example.com`

It continues to check the path rules afterward.

If it's not this Host, it may:

- Not match this rule
- Return default 404
- Or redirect to other matching rules

This also explains why verification uses:

    curl -H "Host: nginx.example.com" http://10.0.0.20:31152

Because Host is very important for Ingress matching.

---

### 10. `http`

This layer means:

> This rule group is for HTTP routing rules.

That is, under the premise of `host: nginx.example.com`, the request should continue to match based on HTTP path.

At this stage, you can simply understand:

- `host` determines which domain
- `http.paths` determines which path under this domain should be forwarded to which Service

---

### 11. `paths`

`paths` means:

> A set of path matching rules defined under the current Host.

Because there may be multiple paths under the same domain, for example:

- `/`
- `/api`
- `/admin`

And each path may correspond to different backends, so `paths` is also a list.

In this minimal example, only one path rule is defined.

---

### 12. `- path: /`

Means:

> This path rule matches the root path `/`

Its meaning isn't just matching "exactly one slash", but should be understood in combination with `pathType`.

Looking at this line alone, you can first understand it as:

- This rule starts matching from `/`

---

### 13. `pathType: Prefix`

This line is very important.

It means:

> The path is processed using "prefix matching".

Current configuration is:

    path: /
    pathType: Prefix

This means:

- `/`
- `/index.html`
- `/abc`
- `/anything`

All these paths can match this rule because they all start with the prefix `/`.

This is also why this minimal example is very suitable as the first Ingress rule:

- The rule is simple
- The coverage is broad
- As long as the Host is correct, most ordinary paths can match

At this stage, just remember:

- `Prefix` = prefix matching
- `/ + Prefix` = most ordinary requests under this domain can match

---

### 14. `backend`

`backend` means:

> After this Host + Path rule matches, traffic should be forwarded to where.

This is the final destination of the entire Ingress YAML.

The previous `host`, `path`, and `pathType` are all answering:

- Which requests can match this rule

While `backend` answers:

- After matching, to which service should the request be forwarded

---

### 15. `backend.service`

This layer means:

> The backend target is a Service.

This is a key understanding at this stage:

> Ingress usually doesn't directly forward traffic to Pods, but first to a Service.

The reason is:

- Pods may change
- Pod IPs may change
- Services provide stable entry points
- Services can hide backend Pod changes

Thus, Ingress and backend workloads are typically separated by a Service.

---

### 16. `backend.service.name: nginx-web-svc`

Means:

> After this rule matches, traffic should be forwarded to the Service named `nginx-web-svc`.

This references the Service created earlier:

    nginx-web-svc

This is why creating the Service first is essential.  
If the name is written incorrectly, for example:

    nginx-web-service

It would result in:

- Ingress rule matches
- But no correct backend found
- May return 502 / 503

So this line is one of the most commonly checked places during troubleshooting.

---

### 17. `backend.service.port.number: 80`

Means:

> Forward traffic to the 80 port of `nginx-web-svc`.

This line doesn't mean:

- The Controller directly finds the Pod's 80 port

But rather:

- First forward traffic to the Service's 80 port
- Then the Service forwards it to the backend Pod based on its configuration

This is also a key point for understanding 502 / 503 errors later:

- If the port here is wrong
- Or the Service's own port definition doesn't match

Even if the Ingress rule matches, traffic will still fail.

---

## IX. Turn the entire Ingress YAML into a complete sentence

You can summarize this YAML as the following sentence:

> Create an Ingress rule named `nginx-web-ingress` in the `test` namespace, and hand it over to the ingress-nginx Controller associated with the IngressClass `nginx`; when the request's Host is `nginx.example.com` and the path matches the prefix `/`, forward the traffic to the backend Service `nginx-web-svc` on port 80.

If you can make sense of this sentence, it means you've truly understood the YAML.

---

## 10. Revisiting the YAML from the Request Perspective

Assume you execute the following command:

    curl -H "Host: nginx.example.com" http://10.0.0.20:31152

The role this YAML plays in the entire process is as follows:

### Step 1: Request arrives at the Controller entry point

Because `10.0.0.20:31152` is actually the NodePort entry point of the ingress-nginx Controller.

### Step 2: Controller receives the HTTP request

The ingress-nginx Controller begins processing the request.

### Step 3: Read the Host header from the request

The Controller sees:

    Host: nginx.example.com

### Step 4: Match the Ingress rule

The Controller looks for rules that satisfy:

- `host = nginx.example.com`
- `path = /`
- `pathType = Prefix`

### Step 5: Match `nginx-web-ingress`

Because the current Ingress exactly defines this rule.

### Step 6: Forward to `nginx-web-svc:80`

The Controller forwards the traffic to the Service based on the backend configuration.

### Step 7: Service looks up Endpoints

`nginx-web-svc` finds the backend Pod based on the selector.

### Step 8: Service forwards traffic to a specific Nginx Pod

For example, one of the `nginx-web` Pods.

### Step 9: Pod returns the page content

The request processing is complete, and the response is returned to the client along the original path.

---

## 11. Applying the Ingress and Checking Status

### Apply command

    kubectl apply -f nginx-web-ingress.yaml

### Check commands

    kubectl get ingress -n test -o wide
    kubectl describe ingress nginx-web-ingress -n test

### Key confirmations

#### 1. Ingress object has been created
Should see:

- `nginx-web-ingress`

#### 2. CLASS is correct
Should see:

- `nginx`

#### 3. HOSTS are correct
Should see:

- `nginx.example.com`

#### 4. BACKEND is correct
In the output of `describe`, should see:

- `nginx-web-svc:80`

#### 5. ADDRESS explanation
In the kubeadm + NodePort scenario, `ADDRESS` may show two situations:

- Initially empty
- Later written back by the ingress-nginx Controller as a node address, e.g., `10.0.0.21`

Both situations are normal.  
At this stage, the more stable entry understanding should still be based on:

    NodeIP:NodePort

rather than relying solely on the column `ADDRESS`.

---

## 12. Why `kubectl get all` Can't See the Ingress

During troubleshooting, a common phenomenon is as follows:

    kubectl get all -n test

Can see:

- Pod
- Service
- Deployment
- ReplicaSet

But cannot see:

- Ingress

This is normal behavior.  
The reason is:

> `kubectl get all` does not display Ingress by default.

To view Ingress explicitly, execute:

    kubectl get ingress -n test

or:

    kubectl get pod,svc,deploy,rs,ing -n test

---

## 13. Configuring hosts

To allow local machines to access the current Ingress via domain name, configure hosts on the local machine.

### Linux / macOS

Edit:

    /etc/hosts

Append:

    10.0.0.20 nginx.example.com

### Windows

Edit:

    C:\Windows\System32\drivers\etc\hosts

Append:

    10.0.0.20 nginx.example.com

### Notes

At this stage, it's recommended to resolve to a reachable node, e.g.,

    10.0.0.20

If access fails later, you can also change it to:

- `10.0.0.21`
- `10.0.0.22`

and continue verification.

---

## 14. Verification Methods

### Method 1: Access via hosts

    curl http://nginx.example.com:31152

### Method 2: Verify without modifying hosts, using the Host header directly

    curl -H "Host: nginx.example.com" http://10.0.0.20:31152

If needed, you can also use:

    curl -H "Host: nginx.example.com" http://10.0.0.21:31152
    curl -H "Host: nginx.example.com" http://10.0.0.22:31152

### Recommended Verification Method at This Stage

Prioritize using:

    curl -H "Host: nginx.example.com" http://10.0.0.20:31152

Reasons:

- No dependency on hosts
- Can directly verify if the Ingress rule is matched
- More suitable for initial troubleshooting

---

## 15. Complete Request Flow Explanation

In the current environment, the request flow should be understood as:

    Client
    -> 10.0.0.20:31152
    -> ingress-nginx Controller
    -> Match Host: nginx.example.com
    -> Match Path: /
    -> Forward to nginx-web-svc:80
    -> nginx-web Pod
    -> Return page content

This flow is the core picture to establish at this stage.

---

## 16. Minimum Check Order When Initial Failures Occur

### 1. Check Controller /think

kubectl get pods -n ingress-nginx  
kubectl get svc -n ingress-nginx  

**Verification:**  
- Controller Pod is normal  
- Controller Service exists  
- NodePort is `31152`  

### 2. Check Business Pod  

    kubectl get pods -n test -o wide  

**Verification:**  
- Nginx Pod is `Running`  
- `READY` is normal  

### 3. Check Service and Endpoints  

    kubectl get svc -n test  
    kubectl get endpoints -n test  

**Verification:**  
- `nginx-web-svc` exists  
- Endpoints is not empty  

### 4. Check Ingress  

    kubectl get ingress -n test -o wide  
    kubectl describe ingress nginx-web-ingress -n test  

**Verification:**  
- `CLASS` is `nginx`  
- `HOSTS` is `nginx.example.com`  
- backend is `nginx-web-svc:80`  

### 5. Check if requests hit the rules  

    curl -H "Host: nginx.example.com" http://10.0.0.20:31152  

---  

## Seventeen. Common Mistakes in Current Environment  

### 1. Still using old example port `30080`  
Correct port in current environment is:  
- HTTP: `31152`  
- HTTPS: `30361`  

### 2. Mistakenly taking ClusterIP as external entry  
For example:  
- `10.106.166.208`  
- `10.99.143.71`  

These addresses are not the main external entry of this stage.  

### 3. Forgetting to write `ingressClassName: nginx`  
This will cause Ingress possibly not being taken over by ingress-nginx.  

### 4. Only executing `kubectl get all`  
This will lead to misunderstanding that Ingress does not exist.  

### 5. Service is normal, but Endpoints is empty  
This will cause Ingress rules hitting but still unable to access backend Pod.  

### 6. Only understanding YAML, not understanding rules from request chain  
This will make it difficult to judge the problem location in Host, Path, or backend Service during troubleshooting.  

---  

## Eighteen. Interview Answer Reference  

If asked "How to do a minimal Ingress experiment", you can answer as follows:  

In a kubeadm self-built cluster, first confirm that ingress-nginx Controller is installed, and confirm the entry is NodePort, for example, HTTP port in current environment is `31152`.  
Then create a Nginx Deployment and corresponding ClusterIP Service, ensuring the backend chain is established.  
Next, create an Ingress, specify `ingressClassName: nginx`, configure Host and Path, forwarding `nginx.example.com` requests to `nginx-web-svc:80`.  
Finally, verify through `curl -H "Host: nginx.example.com" http://NodesIP:31152`, confirming the request passes through Controller, Ingress rules, Service, and finally reaches the backend Pod.  

---  

## Nineteen. Phase Conclusion  

After completing this article, the following conclusions should be formed:  

- ingress-nginx has provided entry through NodePort in current environment  
- HTTP entry port is `31152`  
- HTTPS entry port is `30361`  
- `IngressClass/nginx` exists  
- `nginx-web-ingress` is handed over to ingress-nginx via `ingressClassName: nginx`  
- Current stage business access should use:  

    NodeIP:31152  

as unified entry base  
- `curl -H "Host: ..."` is the most important first-round verification method  
- Key of Ingress YAML is not memorizing fields, but understanding "who matches, who takes over, who forwards, and finally who receives"  

---  

## Keyword Mnemonics  

- Ingress backend: usually points to Service, not directly to Pod  
- ingressClassName: specifies which Ingress Controller takes over  
- NodePort: entry exposure method in current environment  
- Host header: domain information in HTTP request header  
- Endpoints: actual backend address collection of Service  
- `kubectl get all`: default does not display Ingress  
- `pathType: Prefix`: matches by path prefix  

---  

## Next Day Suggestions  

Next article suggestion:  

[[06-Ingress Troubleshooting - Access Failure 404 502 No Backend Traffic]]