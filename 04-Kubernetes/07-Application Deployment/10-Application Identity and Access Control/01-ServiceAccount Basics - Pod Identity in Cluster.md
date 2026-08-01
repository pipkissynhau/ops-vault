# 01-ServiceAccount Basics: Pod Identity in the Cluster

## Document Notes
- Document positioning: Kubernetes Application Identity and Permission Control Introduction
- Applicable stages: 07-Application Deployment / 09-Application Identity and Permission Control
- Learning objectives:
  - Understand why Pods need "identity"
  - Understand the role of `ServiceAccount` in Kubernetes
  - Be able to distinguish between "human access to the cluster" and "Pod access to the cluster" identity types
  - Be able to understand the basic ServiceAccount binding and usage methods
  - Lay the foundation for subsequent learning of RBAC, minimal permissions, and application security

## Build an Intuition First

In the previous application deployment stage, you've learned:

- How Pods are scheduled to nodes
- How applications run through Deployments
- How Services direct traffic to Pods

But after an application runs, another very practical issue arises:

**Who is this Pod in the cluster? Can it access the Kubernetes API? If so, what can it do?**

Examples include:

- A monitoring component needs to read Pod, Node, Service information
- A controller needs to monitor resource changes
- An automation task needs to create Jobs, update ConfigMaps
- A business container calls the Kubernetes API to query its runtime environment

At this point, the Pod cannot "anonymously" perform actions.  
It must have an identity within the cluster.

This identity is `ServiceAccount`.

## What is a ServiceAccount

`ServiceAccount` is the identity object used by **Pods/workloads in Kubernetes**.

You can think of it as:

**The "system account" or "application account" of a Pod in the Kubernetes cluster.**

It is not for human login, but for programs, Pods, and controllers that are running.

In other words:

- User account: for humans
- ServiceAccount: for Pods

This is the first prerequisite for understanding RBAC later.

## Why Pods Need Identity

Because many Pods are not just "running a process" - they may need to interact with the Kubernetes control plane.

Common scenarios include:

### 1. Reading Cluster Resources

Examples include:

- Querying which Pods exist in the current Namespace
- Viewing Services, Endpoints
- Getting ConfigMaps, Secrets
- Reading node information

### 2. Operating Cluster Resources

Examples include:

- Creating Jobs
- Updating Leases
- Deleting certain temporary resources
- Modifying a configuration object

### 3. Being Recognized by Platform Components

Examples include:

- Controllers recognizing the caller's identity
- Audit logs recording who accessed the API
- Permission systems determining if this Pod has the right to operate

Thus, the core role of a ServiceAccount can be summarized as:

**Giving Pods a clear identity when accessing the Kubernetes API.**

## First Distinguish Two Types of Identity

When studying this section, always keep these two things separate.

### 1. User Identity

Examples include:

- You locally executing `kubectl get pods`
- Administrators logging into the platform to operate the cluster
- Operations personnel using kubeconfig to manage resources

This is the "human" identity.

### 2. Pod Identity

Examples include:

- A controller in the cluster accessing the API
- A program inside a container querying Pod information
- An Agent pulling or reporting Kubernetes resources

This is the "program/Pod" identity.

`ServiceAccount` is the second type of identity.

## The Essence of ServiceAccount

You can first think of it as an identity object in Kubernetes, with characteristics including:

- Belongs to a specific Namespace
- Can be referenced by Pods
- Can be bound to permission rules
- Used by Pods to access the Kubernetes API
- Often used together with RBAC

In other words:

**ServiceAccount is only responsible for "who I am", while "what I can do" is determined by RBAC.**

This statement is very important.

## There is a Default ServiceAccount

In each Namespace, Kubernetes automatically creates a ServiceAccount named:

    default

You can check:

    kubectl get sa -n default

Or check a specific namespace:

    kubectl get sa -n kube-system

You'll typically see:

    NAME      SECRETS   AGE
    default   0         10d

This indicates:

**Even if you don't explicitly create a ServiceAccount, Pods usually default to using the `default` ServiceAccount in the current Namespace.**

## The Significance of the Default ServiceAccount

If a Pod doesn't specify `serviceAccountName`, it generally uses the default ServiceAccount in its namespace.

Example:

    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx-test
    spec:
      containers:
        - name: nginx
          image: nginx:1.25

Although this YAML doesn't specify `serviceAccountName`, it's not "without identity" - it's defaulting to:

    serviceAccountName: default

So remember:

**Not writing it doesn't mean there's no identity; it usually means using the default account.**

## Why Not to Rely on the Default ServiceAccount Long-Term

From a security perspective, this approach is often unclear and not conducive to minimal permissions control.

Issues include:

- Different applications sharing the same identity, making auditing difficult
- Ambiguous boundaries when converging permissions later
- If the default account's permissions are expanded, many Pods will be affected
- In production environments, it's hard to distinguish "what permissions each application should have"

Therefore, in standardized environments, it's recommended to:

**Use a dedicated ServiceAccount for each application, controller, and component that needs to access the API.**

## First ServiceAccount Example

Let's create an independent ServiceAccount:

    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: app-reader
      namespace: default

Apply it:

    kubectl apply -f sa-app-reader.yaml

Check:

    kubectl get sa -n default
    kubectl describe sa app-reader -n default

Now you've created an identity object available to Pods, but it's still just "having an identity" without any additional permissions.

## Let Pod Use a Specified ServiceAccount

Pods specify which ServiceAccount they use via the `serviceAccountName` field.

Example:

    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx-with-sa
      namespace: default
    spec:
      serviceAccountName: app-reader
      containers:
        - name: nginx
          image: nginx:1.25

Apply:

    kubectl apply -f pod-with-sa.yaml

Check Pod details:

    kubectl describe pod nginx-with-sa

You'll see output similar to:

    Service Account:  app-reader

This indicates that this Pod's identity in the cluster is no longer the default `default`, but instead `app-reader`.

## What Exactly Does a ServiceAccount Give to a Pod

This is a critical question.

A ServiceAccount itself does not directly grant a Pod many capabilities. It mainly provides:

- Identity declaration
- Authentication context for API access
- Carrier for RBAC permission system linkage

So more accurately:

**ServiceAccount is "identity", RBAC is "permission".**

You can analogize these two as:

- ServiceAccount: ID badge
- RBAC: Access control permissions

Having an ID badge doesn't mean you can enter all doors.  
Whether you can enter depends on access control policies.

## How This Identity Is Reflected in Containers

When a Pod uses a ServiceAccount, Kubernetes often mounts access credentials related to this identity into the container for program access to the Kubernetes API.

Common paths in containers are similar to:

    /var/run/secrets/kubernetes.io/serviceaccount/

Inside you might find:

- token
- namespace
- ca.crt

This means:

**Programs inside the container can use these credentials to access the Kubernetes API Server with the current ServiceAccount identity.**

You can enter the container to check:

    kubectl exec -it nginx-with-sa -- sh

Then check:

    ls /var/run/secrets/kubernetes.io/serviceaccount/

This step mainly helps you build awareness:  
ServiceAccount is not an abstract concept—it actually affects the API access identity context inside containers.

## What Is automountServiceAccountToken

Some Pods do not need to access the Kubernetes API.  
In such cases, automatically mounting the ServiceAccount token increases the attack surface.

Kubernetes provides a control option:

    automountServiceAccountToken: false

Example:

    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx-no-token
    spec:
      serviceAccountName: app-reader
      automountServiceAccountToken: false
      containers:
        - name: nginx
          image: nginx:1.25

This indicates:

- The Pod still uses `app-reader` as its identity
- But does not automatically mount the access token into the container

From a security perspective, this is important.

You can initially remember:

**If an application does not need to access the Kubernetes API, consider disabling token auto-mounting.**

This is part of the foundational practice of "minimum exposure" later on.

## Namespace Boundaries of ServiceAccounts

ServiceAccounts are **namespace-level resources**.

For example, if you create:

    namespace: dev
    name: app-reader

Then it fully means:

**The app-reader in the dev namespace**

It cannot be directly referenced as a local object by Pods in other namespaces.

In other words, Pods can only reference ServiceAccounts in their own namespace.

This helps you understand why RBAC later distinguishes between:

- Role / RoleBinding
- ClusterRole / ClusterRoleBinding

And why many permission controls are strongly namespace-related.

## View Current ServiceAccount Used by a Pod

### View Pod YAML

    kubectl get pod nginx-with-sa -o yaml

Focus on:

    serviceAccountName: app-reader

### View Pod Description Information

    kubectl describe pod nginx-with-sa

### View ServiceAccounts in Namespace

    kubectl get sa -n default

## Using ServiceAccount in Deployment

In actual production, it's more common to specify it in a Deployment rather than a standalone Pod.

Example:

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

This means all Pods created by this Deployment will use the same ServiceAccount.

This is very common in production, for example:

- A controller with a group of Pods sharing a ServiceAccount
- An Agent component with a group of instances sharing a ServiceAccount
- A business service using a unified independent identity

## Common Use Cases

### 1. Controller Class Components

Examples:

- Custom controller
- Operator
- Some automation schedulers

These components typically need access to Kubernetes API, so they must have a dedicated ServiceAccount.

### 2. Monitoring, Logging, Platform Agents

Examples:

- Some monitoring collectors
- Logging components
- Platform inspection agents

They may need to read information about Pods, Nodes, Namespaces, etc.

### 3. Business Applications That Call Kubernetes API

Examples:

- Applications that need to query which Namespace they're running in after startup
- Applications that need to read certain resources in the cluster
- Applications that perform automated resource management

### 4. Security Isolation and Auditing

Even if multiple applications need minimal API permissions, it's recommended to split them into different ServiceAccounts to avoid sharing the default identity.

## Common Misconceptions

### Misconception 1: ServiceAccount Is Permission

No.  
ServiceAccount is just an identity, not permission itself.

The actual permission is determined by the RBAC binding that follows.

### Misconception 2: Creating a ServiceAccount Automatically Grants Access to All Resources

No.  
Without corresponding Role / ClusterRole bindings, it typically can't perform any sensitive operations.

### Misconception 3: Pods Without serviceAccountName Have No Identity

No.  
Most of the time, it will use the `default` ServiceAccount in the current Namespace.

### Misconception 4: All Pods Should Mount Token

No.  
If the application doesn't access Kubernetes API, consider disabling automatic mounting to reduce exposure.

## A More Production-Ready Understanding

You can think of it this way:

- Whether a Pod can be scheduled to a node is the scheduling problem discussed earlier
- Who the Pod is in the cluster after it runs is the ServiceAccount problem
- Whether the Pod can read Pod lists, modify ConfigMap, or view Secret is the RBAC problem

Thus, the entire "application security operation" chain is:

1. First, get the Pod running
2. Then assign it a clear identity
3. Then grant only the necessary permissions

This connects to the "principle of least privilege" you'll learn later.

## Troubleshooting Approach: Which ServiceAccount Does a Pod Use

If you suspect a Pod has the wrong identity, check in this order.

### Step 1: Check Pod Definition

    kubectl get pod <pod-name> -o yaml

Check:

    serviceAccountName

### Step 2: Check Pod Description

    kubectl describe pod <pod-name>

### Step 3: Check if the Target ServiceAccount Exists

    kubectl get sa -n <namespace>

### Step 4: Check for Accidental Use of default

Many issues aren't "no identity", but rather "unexpected use of default".

### Step 5: Check for Automatic Token Mounting

If the application shouldn't access API but has a token mounted, check:

- Whether the Pod has set `automountServiceAccountToken: false`
- Whether the ServiceAccount itself has relevant settings

## A Minimal Experiment Recommendation

### Experiment Objective

Understand how a Pod uses an independent ServiceAccount and observe the ServiceAccount-related mounts in the container.

### Step 1: Create a ServiceAccount

    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: demo-sa
      namespace: default

Apply:

    kubectl apply -f demo-sa.yaml

### Step 2: Create a Pod Using This ServiceAccount

    apiVersion: v1
    kind: Pod
    metadata:
      name: demo-sa-pod
      namespace: default
    spec:
      serviceAccountName: demo-sa
      containers:
        - name: nginx
          image: nginx:1.25

Apply:

    kubectl apply -f demo-sa-pod.yaml

### Step 3: Verify the Pod's Identity

    kubectl describe pod demo-sa-pod

### Step 4: Enter the Container to Check Mounted Directories

    kubectl exec -it demo-sa-pod -- sh

Check:

    ls /var/run/secrets/kubernetes.io/serviceaccount/
    cat /var/run/secrets/kubernetes.io/serviceaccount/namespace

Through this experiment, you'll gain a clearer understanding of:

- ServiceAccount is genuinely used by the Pod
- The container can see credentials related to the ServiceAccount mounted
- The Pod isn't an "anonymous process" in the cluster

## Key Points Recap

Remember these core points:

1. `ServiceAccount` is the identity object used by Pods / applications  
2. It answers the question of "who the Pod is in the cluster"  
3. When unspecified, Pods typically use the `default` ServiceAccount in the current Namespace  
4. `ServiceAccount` handles identity, RBAC handles permissions  
5. Pods specify which identity to use via `serviceAccountName`  
6. Workloads needing Kubernetes API access should typically use an independent ServiceAccount  
7. Pods not needing API access should consider disabling token auto-mounting  
8. Long-term abuse of `default` ServiceAccount as a unified identity for all applications is not recommended  

## Common Commands Quick Reference

    kubectl get sa
    kubectl get sa -n default
    kubectl describe sa app-reader -n default
    kubectl apply -f sa.yaml
    kubectl apply -f pod.yaml
    kubectl get pod -o yaml
    kubectl describe pod <pod-name>
    kubectl exec -it <pod-name> -- sh

## One-Sentence Summary

`ServiceAccount` essentially answers the question: **Who is this Pod in the Kubernetes cluster?**

## Tags
#Kubernetes #ApplyDeployment #ServiceAccount #PodIdentity #RBAC #MinimumPermissions #ApplicationSecurity

## Operational Extension Understanding
- Kubernetes Official Documentation: Service Accounts  
  https://kubernetes.io/docs/concepts/security/service-accounts/
- Kubernetes Official Documentation: Configure Service Accounts for Pods  
  https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/
- Kubernetes Official Documentation: Using RBAC Authorization  
  https://kubernetes.io/docs/reference/access-authn-authz/rbac/

## Next Day Plan
- Study [[02-RBAC Basics - Role ClusterRole RoleBinding ClusterRoleBinding]]
- Understand the difference between "having identity" and "having permissions"
- Establish foundational understanding of Kubernetes application permission control