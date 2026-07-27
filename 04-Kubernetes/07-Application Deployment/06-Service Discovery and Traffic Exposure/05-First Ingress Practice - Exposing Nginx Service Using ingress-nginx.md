# 05-First Ingress Practice: Exposing Nginx Service Using ingress-nginx

## Document Description
- Document Purpose: This is the first comprehensive practical example of learning about Ingress in the context of Kubernetes service discovery and traffic routing.
- Suitable for: Those who have understood the role of Ingress, the basic principles of the Ingress Controller, and have completed the installation of the ingress-nginx Controller and prepared NodePort entries in a kubeadm cluster. This guide helps you establish a basic entry-linkage mechanism through a minimal example.
- Recommended Reading Path: `04-Kubernetes/07-Application Deployment/06-Service Discovery and Traffic Routing/05-First Ingress Practice: Exposing Nginx Service.md`

## Tags
#Kubernetes #Ingress #IngressController #ingress-nginx #Nginx #Service #NodePort #HTTP #Domain Name #Path Forwarding #kubeadm #Application Deployment #Cloud Native #Operation and Maintenance #Interview Notes

---

## I. Objectives of This Document

Based on the actual environment in the current kubeadm cluster, this document aims to complete a minimal Ingress experiment that exposes the Nginx service through ingress-nginx.

After completing this document, you should be able to:

- Deploy a minimal Nginx Deployment.
- Create a ClusterIP Service as the Ingress backend.
- Create an Ingress managed by ingress-nginx.
- Understand the entire process by which requests enter the Controller via `NodeIP:NodePort` and are then forwarded to the backend Service and Pods.
- Use `curl -H "Host: ..."` to verify whether the Ingress is functioning correctly.
- Gain a detailed understanding of the meaning of each key field in the Ingress YAML configuration.

---

## II. Current Environment Information

The current cluster nodes are as follows:

- `k8s-master`: `10.0.0.20`
- `k8s-node01`: `10.0.0.21`
- `k8s-node02`: `10.0.0.22`

The current entry points for the `ingress-nginx-controller` are:

- HTTP: `31152`
- HTTPS: `30361`

The current `IngressClass` configuration is:

- `nginx -> k8s.io/ingress-nginx`

Therefore, at this stage, the entry points can be understood as follows:

### HTTP Entry Points
    http://10.0.0.20:31152
    http://10.0.0.21:31152
    http://10.0.0.22:31152

### HTTPS Entry Points
    https://10.0.0.20:30361
    https://10.0.0.21:30361
    https://10.0.0.22:30361

---

## III. Experimental Objectives

The objectives of this experiment are as follows:

- Deploy a business Deployment named `nginx-web`.
- Create a ClusterIP Service named `nginx-web-svc`.
- Create an Ingress named `nginx-web-ingress`.
- Ensure that requests to `nginx.example.com` are forwarded through the ingress-nginx Controller to `nginx-web-svc:80`.

The expected access method is:

    http://nginx.example.com:31152

Alternatively, you can verify it without changing the hosts configuration by using the Host header:

    curl -H "Host: nginx.example.com" http://10.0.0.20:31152

---

## IV. Prerequisites

Before creating the business Ingress, the following prerequisites must be met:

### 1. The ingress-nginx Controller is installed and running normally

Execute the following commands:

    kubectl get pods -n ingress-nginx
    kubectl get svc -n ingress-nginx
    kubectl get ingressclass

You should confirm at least the following:

- `ingress-nginx-controller-xxxxx` is in the `Running` state.
- The `ingress-nginx-controller` uses a `NodePort`.
- The HTTP NodePort is `31152`, and the HTTPS NodePort is `30361`.
- The `IngressClass/nginx` exists.

### 2. It is confirmed that the current environment entry point is not the business Service but the Controller's NodePort

That is:

- Clients first access `NodeIP:31152`.
- Then, the ingress-nginx Controller forwards requests to the backend business Service according to Ingress rules.

---

## V. Deploying the Backend Business: Nginx Deployment

### Deployment YAML

    apiVersion: apps/v              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: nginx-web-svc
                    port:
                      number: 80

### The Overall Meaning of This YAML

The function of this Ingress can be understood as follows:

- When a request enters the ingress-nginx Controller,
- and the Host in the request is `nginx.example.com`,
- and the path matches `/`,
- the traffic will be forwarded to `nginx-web-svc:80` in the `test` namespace.

Essentially, it is not about "creating an entry program," but rather "declaring a rule for forwarding requests."

In other words:

- The ingress-nginx Controller is responsible for receiving requests,
- and the Ingress YAML tells the Controller to which target these requests should be directed.

---

## Section 8: Detailed Explanation of Each Field in the Ingress YAML

This section is particularly important because it marks your first experience writing Ingress rules as configuration.

### 1. `apiVersion: networking.k8s.io/v1`

This indicates the API group and version to which the current resource object belongs.

Here, it means that:

- This resource is part of the `networking.k8s.io` API group in Kubernetes,
- and it uses the `v1` version.

For now, you can think of it as:

> A declaration of the version used by Kubernetes to identify this Ingress resource configuration.

---

### 2. `kind: Ingress`

This specifies that the resource being created in this YAML is an **Ingress**.

This line determines how Kubernetes will process this configuration. By specifying `Ingress`, it means that:

> This is not a workload object or a service object, but rather an entry rule object.

---

### 3. `metadata`

`metadata` contains basic information about the resource itself.

In this YAML, the most important fields are:

- `name`
- `namespace`

---

### 4. `metadata.name: nginx-web-ingress`

This sets the name of the Ingress rule object to `nginx-web-ingress`.

Its main purpose is:

- To make it easy to identify and manage this rule,
- and later to view its details using commands like:

    kubectl describe ingress nginx-web-ingress -n test

The name is used by Kubernetes for internal management purposes, not for client access.

---

### 5. `metadata.namespace: test`

This specifies that the Ingress resource is created in the `test` namespace.

This is crucial because the Service referenced by the Ingress backend should also be in the same namespace.

In other words:

- If this Ingress is in the `test` namespace,
- then the `nginx-web-svc` it references must also be in `test`.

If the namespaces do not match, issues such as:

- The Ingress being created successfully but the backend Service not being found

may occur. Therefore, it is a good practice to ensure that Ingresses, backend Services, and related Pods are all placed in the same namespace.

---

### 6. `spec`

`spec` contains the core details of the Ingress rule.

You can think of it as:

- `metadata` defines who this rule is for and where it is located,
- while `spec` specifies how traffic should be forwarded according to this rule.

The most critical fields are included in the `spec` section.

---

### 7. `spec.ingressClassName: nginx`

This specifies that the Ingress rule should be processed by the Controller associated with the `nginx` IngressClass.

In the current environment:

- The name of the `IngressClass` is `nginx`,
- and its corresponding Controller is `k8s.io/ingress-nginx`.

Therefore, setting `ingressClassName: nginx` ensures that the ingress-nginx controller will handle this rule.

This field is crucial because:

- If it is set incorrectly,
- for example, to a non-existent class or a class that does not match the existing one in the cluster,
- the Ingress may be created successfully but not properly processed.

For now, remember that:

> `ingressClassName` determines which Controller will manage this rule.

---

### 8. `spec.rules`

`rules` defines a set of entry rules for this Ingress.

A single Ingress can have multiple rules, covering different hosts, paths, or backends. In this basic example, only one rule is defined, so there is only one entry starting with `-`.

You can think of `rules` as:

> A collection of routing rules for the Ingress.

---

### 9. `- host: nginx.example.com`

This rule applies only to requests with a Host of `nginx.example.com`.

This is an important aspect of Ingress rules, as they not only consider IP addresses and ports but also the Host field in HTTP request headers.

In this context, it means that if a request enters the Controller and its Host is `nginx.example.comEven if the Ingress rule is matched, the traffic will still fail.

---

## IX. Combining the Entire Ingress YAML into a Complete Sentence

This YAML can be summarized as follows:

> Create an Ingress rule named `nginx-web-ingress` in the `test` namespace and assign it to the ingress-nginx Controller corresponding to the `nginx` IngressClass; when the requested Host is `nginx.example.com` and the path matches with the prefix `/`, the traffic will be forwarded to port 80 of the backend Service `nginx-web-svc`.

If you can articulate this sentence smoothly, it means you have truly understood this YAML.

---

## X. Revisiting This YAML from the Request Perspective

Assume you execute the following command:

    curl -H "Host: nginx.example.com" http://10.0.0.20:31152

In this entire process, the role of this YAML is as follows:

### Step 1: The request first reaches the Controller entrance

Because `10.0.0.20:31152` is actually the NodePort entrance of the ingress-nginx Controller.

### Step 2: The Controller receives the HTTP request

The ingress-nginx Controller begins processing this request.

### Step 3: Read the Host from the request header

The Controller sees:

    Host: nginx.example.com

### Step 4: Match the Ingress rule

The Controller checks whether there is a rule that meets the following criteria:

- `host = nginx.example.com`
- `path = /`
- `pathType = Prefix`

### Step 5: The rule `nginx-web-ingress` is matched

Because this Ingress rule is precisely defined.

### Step 6: Forward the traffic to `nginx-web-svc:80`

The Controller forwards the traffic to the Service based on the backend configuration.

### Step 7: The Service searches for Endpoints

`nginx-web-svc` finds the backend Pod according to the selector.

### Step 8: The Service forwards the traffic to a specific Nginx Pod

For example, one of the `nginx-web` Pods.

### Step 9: The Pod returns the page content

The request processing is completed, and the response is sent back to the client via the original path.

---

## XI. Applying Ingress and Checking Its Status

### Application command

    kubectl apply -f nginx-web-ingress.yaml

### Check command

    kubectl get ingress -n test -o wide
    kubectl describe ingress nginx-web-ingress -n test

### Key confirmations

#### 1. The Ingress object has been created
You should see:

- `nginx-web-ingress`

#### 2. The CLASS is correct
You should see:

- `nginx`

#### 3. THE HOSTS are correct
You should see:

- `nginx.example.com`

#### 4. THE BACKEND is correct
In the `describe` output, you should see:

- `nginx-web-svc:80`

#### 5. Explanation of ADDRESS
In a kubeadm + NodePort scenario, the `ADDRESS` may have two possible values:

- Initially empty
- Later rewritten by the ingress-nginx Controller to a specific node address, such as `10.0.0.21`

Both situations are normal. At this stage, it is more reliable to refer to:

    NodeIP:NodePort

rather than relying solely on the `ADDRESS` column.

---

## XII. Why Can't You See Ingress with `kubectl get all`?

During troubleshooting, a common scenario is:

    kubectl get all -n test

You can see:

- Pods
- Services
- Deployments
- ReplicaSets

But you cannot see:

- Ingress

This is normal behavior.  
The reason is:

> `kubectl get all` does not display Ingress by default.

If you want to view Ingress as well, you should explicitly execute:

    kubectl get ingress -n test

Or:

    kubectl get pod,svc,deploy,rs,ing -n test

---

## XIII. Configuring hosts

To allow your local machine to access the current Ingress via a domain name, you can configure hosts on your local system.

### Linux / macOS

Edit:

    /etc/hosts

Add:

    10.0.0.20 nginx.example.com

### Windows

Edit:

    C:\Windows\System32\drivers\etc\hosts

Add:

    10.0.0.20 nginx.example.com

### Note

At this stage, it is recommended to resolve it to a reachable node first, for example:

    10.0.0.20

If access still fails later, you can also try:

- `10.0.0.21`
- `10.0.0.2Next, create an Ingress and specify `ingressClassName: nginx`. Configure the Host and Path to forward requests for `nginx.example.com` to `nginx-web-svc:80`. Finally, verify this by using `curl -H "Host: nginx.example.com" http://nodeIP:31152` to confirm that the requests go through the Controller, Ingress rules, Service, and ultimately reach the backend Pod.

---

## Section 19: Phase Conclusion

After completing this document, the following conclusions should be reached:

- In the current environment, ingress-nginx provides an entry point via NodePort.
- The HTTP entry port is `31152`, and the HTTPS entry port is `30361`.
- The `IngressClass/nginx` already exists.
- `nginx-web-ingress` is managed by ingress-nginx through `ingressClassName: nginx`.
- For current phase, business access should use:

    nodeIP:31152

    as the unified entry point.

- The command `curl -H "Host: ..."` is the most important first-step verification method.
- The key to understanding Ingress YAML is not memorizing fields but comprehending who matches, who takes over, who forwards, and ultimately to whom the requests are directed.

---

## Keywords Summary

- Ingress backend: Usually points to a Service, not directly to a Pod.
- ingressClassName: Specifies which Ingress Controller will manage it.
- NodePort: The method used in the current environment to expose entries.
- Host header: The domain name information in HTTP request headers.
- Endpoints: The actual set of backend addresses for a Service.
- `kubectl get all`: Does not display Ingress by default.
- `pathType: Prefix`: Matches based on the prefix of the path.

---

## Next Day's Suggestions

For the next article, it is recommended to organize the following content:

[[06-Ingress Common Issues Troubleshooting: Access Issues, 404 Errors, 502 Errors, and No Traffic to the Backend]]