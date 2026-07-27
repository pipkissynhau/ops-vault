# 01-ServiceAccount Basics: The Identity of Pods in a Cluster

## Document Description
- Documentation Location: Introduction to Kubernetes Application Identity and Access Control
- Applicable Phase: 07-Application Deployment / 09-Application Identity and Access Control
- Learning Objectives:
  - Understand why Pods also need an "identity"
  - Comprehend the role of `ServiceAccount` in Kubernetes
  - Distinguish between two types of identities: "human access to the cluster" and "Pod access to the cluster"
  - Master the basic methods of binding and using ServiceAccounts
  - Lay a foundation for subsequent studies on RBAC, least privilege, and application security

## Establish an Intuition First

During the application deployment phase, you have already learned:

- How Pods are scheduled onto nodes
- How applications are launched via Deployments
- How Services direct traffic to Pods

However, once an application is running, another practical issue arises:

**Who exactly is this Pod in the cluster? Can it access the Kubernetes API? If so, what can it do?**

For example:

- A monitoring component may need to retrieve information about Pods, Nodes, and Services
- A controller might require monitoring resource changes
- An automated task could need to create Jobs or update ConfigMaps
- A business container might call the Kubernetes API to check its running environment

At this point, a Pod cannot operate "anonymously."  
It must have an identity within the cluster.

This identity is the `ServiceAccount`.

## What is a ServiceAccount?

A `ServiceAccount` is an identity object in Kubernetes that is **used by Pods/workloads**.

You can think of it as:

**The “system account” or “application account” of a Pod within the Kubernetes cluster.**

It is not designed for human login but rather for programs, Pods, and controllers that are running within the cluster.

In other words:

- User Accounts: For humans to use
- ServiceAccounts: For Pods to use

This is the first key concept before understanding RBAC later on.

## Why Do Pods Need an Identity?

Many Pods are more than just running a single process; they may also need to interact with the Kubernetes control plane.

Common scenarios include:

### 1. Reading Cluster Resources

For example:

- Querying which Pods exist in the current Namespace
- Checking Services and Endpoints
- Accessing ConfigMaps and Secrets
- Retrieving node information

### 2. Manipulating Cluster Resources

For example:

- Creating Jobs
- Updating Lease settings
- Deleting temporary resources
- Modifying configuration objects

### 3. Being Recognized by Platform Components

For example:

- Controllers verifying the identity of callers
- Audit logs recording who is accessing APIs
- Permission systems determining whether a Pod has the necessary permissions

Therefore, the core purpose of ServiceAccounts can be summarized as:

**Providing Pods with a clear identity when accessing the Kubernetes API.**

## Distinguish Between Two Types of Identities First

When studying this topic, it is essential to distinguish between these two types of identities:

### 1. User Identity

For example:

- You locally execute `kubectl get pods`
- An administrator logs in to the platform to manage the cluster
- Ops personnel use kubeconfig to manage resources

This is the identity of "humans".

### 2. Pod Identity

For example:

- A controller within the cluster accesses the API
- A program inside a container queries Pod information
- An Agent retrieves or reports Kubernetes resources

This is the identity of "programs/Pods".

A `ServiceAccount` represents the second type of identity.

## The Essence of ServiceAccounts

You can initially understand it as an identity object in Kubernetes, with the following characteristics:

- It belongs to a specific Namespace
- It can be referenced by Pods
- It can be associated with permission rules
- It is used by Pods to access the Kubernetes API
- It is often used together with RBAC

In other words:

**ServiceAccounts are solely responsible for indicating "who I am"; what I can do is determined by RBAC.**

This point is crucial.

## Default ServiceAccounts Exist by Default

In every Namespace, Kubernetes automatically creates a default ServiceAccount named:

    default

You can check this by running:

    kubectl get sa -n default

Or for a specific namespace:

    kubectl get sa -n kube-system

Typically, you will see something like this:

    NAME      SECRETS   AGE
    default   0         10d

This indicates that:

**Even if you do not explicitly create a ServiceAccount, Pods usually use the `default` ServiceAccount in their namespace by default.**

## The Significance of Default ServiceAccounts

If a Pod does not specify a `serviceAccountName`, it generally uses the default ServiceAccount of its namespace.

For example:

    apiVersion: v1
    kind: Pod
    metadata:
      name:### Specification:
```yaml
serviceAccountName: app-reader
automountServiceAccountToken: false
containers:
  - name: nginx
    image: nginx:1.25
```
This indicates that:

- The Pod still uses the `app-reader` identity.
- However, the access token is not automatically mounted inside the container.

From a security perspective, this is very important.

You can keep in mind that:

**If an application does not need to access the Kubernetes API, you should consider disabling automatic token mounting.**

This is a fundamental practice of "minimizing exposure."

## The Scope of ServiceAccounts

ServiceAccounts are **namespace-level resources**.

For example, if you create one with the following settings:

```yaml
namespace: dev
name: app-reader
```

then it specifically belongs to the **`dev` namespace**. It cannot be directly referenced by Pods in other namespaces as a local resource.

In other words, a Pod can only use a ServiceAccount from its own namespace.

This helps you understand why Role-Based Access Control (RBAC) concepts like **Role/RoleBinding** and **ClusterRole/ClusterRoleBinding** are domain-specific to namespaces.

## Checking Which ServiceAccount a Pod Is Using

### Checking the Pod YAML
```bash
kubectl get pod nginx-with-sa -o yaml
```
Look for:

`serviceAccountName: app-reader`

### Checking the Pod Description
```bash
kubectl describe pod nginx-with-sa
```

### Checking the ServiceAccounts in the Namespace
```bash
kubectl get sa -n default
```

## Using ServiceAccounts in Deployments

In production, it is more common to specify ServiceAccounts within a **Deployment** rather than individually for each Pod.

For example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      serviceAccountName: app-reader
      containers:
        - name: nginx
          image: nginx:1.25
```

In this way, all Pods created by the Deployment will use the same `app-reader` ServiceAccount.

This is a common practice in production scenarios:

- A group of controller Pods share one ServiceAccount.
- A set of Agent components use the same ServiceAccount.
- Different business services may have their own independent identities.

## Common Use Cases

### 1. Controller Components
- Custom controllers
- Operators
- Automated schedulers

These components usually need to access the Kubernetes API, so they require a dedicated ServiceAccount.

### 2. Monitoring, Logging, and Platform Agents
- Monitoring collectors
- Log management tools
- System inspection agents

These services may need to retrieve information about Pods, Nodes, namespaces, etc.

### 3. Business Applications That Access the Kubernetes API
- Applications that need to determine their own namespace.
- Applications that require access to cluster resources.
- Applications involved in automated resource management.

### 4. Security Isolation and Auditing
- Even if multiple applications require limited API access, it is recommended to use separate ServiceAccounts to avoid sharing a default identity.

## Common Misconceptions

### Misconception 1: ServiceAccount = Permission
No. A ServiceAccount only defines an identity; permissions are determined by RBAC bindings.

### Misconception 2: Creating a ServiceAccount Automatically Grants Access to All Resources
Not necessarily. Without the right Role/ClusterRole bindings, a ServiceAccount usually cannot perform any sensitive operations.

### Misconception 3: Pods Without a `serviceAccountName` Have No Identity
In most cases, Pods will use the default ServiceAccount of their namespace if no specific one is specified.

### Misconception 4: All Pods Should Use Tokens
Not always. If an application does not need to access the Kubernetes API, disabling automatic token mounting can reduce exposure risks.

## A Practical Understanding

You can think of it this way:

- Whether a Pod can be scheduled onto a node depends on scheduling mechanisms.
- What identity a Pod has within the cluster is determined by its ServiceAccount.
- Whether a Pod can access certain resources (e.g., read ConfigMaps, access Secrets) is controlled by RBAC.

Therefore, the entire process of **secure application operation** involves:

1. First, starting the Pod.
2. Then, assigning it a specific identity through a ServiceAccount.
3. Finally, granting it only the necessary permissions through RBAC.

This leads us to the concept of **minimum permission practice**, which we will discuss later.

## Troubleshooting: Which ServiceAccount Is Being Used by a Pod

If you suspect that a Pod is using the wrong identity, follow these steps:

### Step 1: Check the Pod Definition
```bash
kubectl get pod <pod-name> -o yaml
    kubectl exec -it <pod name> -- sh

## In one sentence

`ServiceAccount` essentially addresses the question: **Who is this Pod within the Kubernetes cluster?**

## Tags
#Kubernetes #Application Deployment #ServiceAccount #Pod Identity #RBAC #Minimum Privilege #Application Security

## Operational Insights
- Kubernetes Official Documentation: Service Accounts  
  https://kubernetes.io/docs/concepts/security/service-accounts/
- Kubernetes Official Documentation: Configuring Service Accounts for Pods  
  https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/
- Kubernetes Official Documentation: Using RBAC Authorization  
  https://kubernetes.io/docs/reference/access-authn-authz/rbac/

## Next Steps
- Study [[02-RBAC Basics: Role, ClusterRole, RoleBinding, ClusterRoleBinding]]
- Understand the difference between "having an identity" and "having permissions"
- Build a foundational understanding of Kubernetes application permission control