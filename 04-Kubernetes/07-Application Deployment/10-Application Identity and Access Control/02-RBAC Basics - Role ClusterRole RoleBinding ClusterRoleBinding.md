# 02-RBAC Basics: Role, ClusterRole, RoleBinding, ClusterRoleBinding

## Document Notes
- Document Location: Kubernetes Application Identity and Permission Control Basics
- Applicable Stage: 07-Application Deployment / 09-Application Identity and Permission Control
- Learning Objectives:
  - Understand that "identity" and "permission" are separate concepts in Kubernetes
  - Understand how RBAC controls what actions users can perform
  - Master `Role`, `ClusterRole`, `RoleBinding`, `ClusterRoleBinding` basic functions
  - Be able to read basic RBAC YAML
  - Lay the foundation for subsequent learning of minimal permission practices

## Build an Intuition First

In the previous lesson we learned `ServiceAccount`.

The next lesson addresses:

**Who is the Pod in the cluster.**

But identity alone is insufficient.  
Because in Kubernetes, the truly important questions also include:

- Can it view Pods
- Can it view ConfigMaps
- Can it create Jobs
- Can it delete resources
- Can it read Secrets
- Can it operate the entire cluster

At this point, a set of "authorization rules" is needed to answer:

**What actions is this identity allowed to perform.**

This mechanism is called RBAC.

## What is RBAC

RBAC is:

**Role-Based Access Control**

In Chinese it's commonly called:

**Based on Role Access Control**

It's the most commonly used authorization model in Kubernetes, used to control:

**Who can perform what actions on which resources.**

You can think of it as:

- `ServiceAccount` answers "Who are you"
- `RBAC` answers "What can you do"

These two concepts must be kept separate.

## Why Kubernetes Needs RBAC

Because Kubernetes is a platform with many resources and operations.

The cluster contains many objects:

- Pods
- Deployments
- Services
- ConfigMaps
- Secrets
- Namespaces
- Nodes
- Jobs
- Ingress
- PersistentVolumes
- CustomResources

Different identities have completely different needs for these resources.

For example:

- A log agent only needs to read Pods and Nodes
- A controller needs to watch/list/update certain resources
- A backup component may need to read PVCs, PVs, and Pods
- A business application may only need to read ConfigMaps in its own namespace
- Some high-privilege components may even need to manage entire cluster resources

Without permission control, two extremes would occur:

- Too much permission, high security risks
- Too little permission, applications can't function

So RBAC's goal is:

**Give only the permissions needed to complete the task, no more.**

## Core Issues of RBAC

You can think of RBAC as answering these three questions:

### 1. Who
Who is operating the resources?

For example:

- A certain `ServiceAccount`
- A certain user
- A certain user group

### 2. What Resources
What objects is it operating on?

For example:

- pods
- configmaps
- secrets
- deployments
- nodes

### 3. What Actions
What operations can it perform?

For example:

- get
- list
- watch
- create
- update
- patch
- delete

So RBAC essentially is a combination authorization model of:

**Subject + Resource + Action**

## Remember the Four Core Objects First

The four most important objects in Kubernetes RBAC learning are:

- `Role`
- `ClusterRole`
- `RoleBinding`
- `ClusterRoleBinding`

You can initially remember them as two dimensions:

### Permission Rule Objects
- `Role`
- `ClusterRole`

### Binding Objects
- `RoleBinding`
- `ClusterRoleBinding`

In other words:

- The first two define "what permissions exist"
- The last two define "who gets these permissions"

## What is a Role

`Role` is used to define:

**A set of permission rules within a namespace.**

It is **namespace-level**.

That means, Role is typically used to describe:

- Within a certain Namespace
- What resources can be accessed
- What actions can be performed

For example:

- Allow reading Pods in the default namespace
- Allow viewing ConfigMaps in a certain namespace
- Allow updating Leases in a certain namespace

You can think of Role as:

**A permission list within a namespace.**

## What is a ClusterRole

`ClusterRole` is used to define:

**A set of permission rules at the cluster level.**

Its main difference from Role is its larger scope.

Common scenarios include:

- Accessing cluster-level resources, such as Nodes, PersistentVolumes, Namespaces
- Defining reusable general permission templates for multiple namespaces
- Granting permissions to cluster-wide controllers

You can think of ClusterRole as:

**Permission rules for the entire cluster, or a globally reusable permission template.**

## Core Differences Between Role and ClusterRole

### Role
- Namespace-level
- Can only describe permissions within a specific namespace

### ClusterRole
- Cluster-level
- Can describe permissions for cluster-wide resources
- Can also be bound to a specific namespace for use

This last point is important:

**ClusterRole does not necessarily mean "permissions are always cluster-wide".**  
It can also be bound to a specific namespace via `RoleBinding`, making it effective only in that namespace.

## What is a RoleBinding

`RoleBinding` serves to:

**Bind a Role or ClusterRole to a subject, effective within a namespace.**

That is, it answers:

**Who gets these permissions.**

This "who" can be:

- `ServiceAccount`
- A user
- A user group

In your current learning path, the most common is binding to `ServiceAccount`.

## What is a ClusterRoleBinding

`ClusterRoleBinding` serves to:

**Bind a ClusterRole to a subject, effective across the entire cluster.**

This means:

- Binding is to a ClusterRole
- Effective scope is the entire cluster

So it's typically used for:

- Cluster-level controllers
- Components needing access to Node and other cluster resources
- System components working across multiple namespaces

## One-Sentence Memory for the Four Objects

You can remember them like this:

- `Role`: Permission rules within a namespace
- `ClusterRole`: Permission rules for the entire cluster or a general permission template
- `RoleBinding`: Bind permissions to a subject, effective within a namespace
- `ClusterRoleBinding`: Bind cluster-level permissions to a subject, effective across the entire cluster

## Common Resources and Actions in RBAC

### Common Resources

Examples:

- `pods`
- `services`
- `configmaps`
- `secrets`
- `deployments`
- `jobs`
- `nodes`
- `namespaces`

### Common Action Verbs

Examples:

- `get`: Read a single object
- `list`: View a list
- `watch`: Continuously monitor changes
- `create`: Create
- `update`: Full update
- `patch`: Partial modification
- `delete`: Delete

You'll soon find that the basic read permissions for many components are typically:

- `get`
- `list`
- `watch`

These three combinations are particularly common.

## First Basic Example: Role

Define a Role that allows reading Pods in a specific namespace.

    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      name: pod-reader
      namespace: default
    rules:
      - apiGroups: [""]
        resources: ["pods"]
        verbs: ["get", "list", "watch"]

This YAML means:

- In the `default` namespace
- Define a Role named `pod-reader`
- It allows performing:
  - `get`
  - `list`
  - `watch` on `pods`

### Understanding apiGroups

Here it writes:

    apiGroups: [""]

This indicates the core resource group, i.e., core API group.  
Resources like these typically belong to the core group:

- pods
- services
- configmaps
- secrets
- namespaces

For Deployments, it would typically write:

    apiGroups: ["apps"]

Just remember this impression for now.

## Second Basic Example: RoleBinding

Having only a Role is insufficient, as it's just the rule itself.  
You also need to bind it to an identity.

For example, bind the above `pod-reader` to the ServiceAccount `app-reader`:

    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: bind-pod-reader
      namespace: default
    subjects:
      - kind: ServiceAccount
        name: app-reader
        namespace: default
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: Role
      name: pod-reader

This YAML means:

- In the `default` namespace
- Bind the `pod-reader` permissions
- To the ServiceAccount `app-reader`

This way, Pods using `app-reader` gain the ability to read Pods in the `default` namespace.

## How to Understand This Minimal Chain

By now, you should be able to connect the chain:

### Step 1: Create ServiceAccount
Solve "Who am I?"

### Step 2: Create Role
Define "What can I do?"

### Step 3: Create RoleBinding
Grant "These permissions to me"

This is the most basic Kubernetes application authorization model.

## Third Basic Example: ClusterRole

Now look at a ClusterRole example.

Assume a component needs to read node information across the entire cluster:

    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: node-reader
    rules:
      - apiGroups: [""]
        resources: ["nodes"]
        verbs: ["get", "list", "watch"]

This ClusterRole means:

- Allow reading Node resources
- This is a cluster-level resource access scenario

Since `nodes` is not a regular namespace resource but a more global scope resource, ClusterRole is typically used.

## Fourth Basic Example: ClusterRoleBinding

If you want to bind this `node-reader` to a ServiceAccount and make it effective globally, you can write:

    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: bind-node-reader
    subjects:
      - kind: ServiceAccount
        name: node-agent
        namespace: kube-system
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: node-reader

This YAML means:

- Bind the `node-reader` ClusterRole
- To the ServiceAccount `node-agent` in the `kube-system` namespace
- And make it effective across the entire cluster

This approach is common in:

- Node monitoring components
- Platform Agents
- Cluster-level system components

## RoleBinding Can Also Bind ClusterRole

This is a particularly confusing but important point.

`RoleBinding` can not only bind to `Role` but also to `ClusterRole`.

### What Does This Mean

It means:

**I want to reuse a globally defined permission rule, but only make it effective in a specific namespace.**

For example:

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: bind-clusterrole-in-default
  namespace: default
subjects:
  - kind: ServiceAccount
    name: app-reader
    namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view

This indicates:

- Using a ClusterRole as a permission template
- But granting permissions only to `default` in this namespace `app-reader`

So you need to distinguish clearly:

- Check "what object is being bound" is Role or ClusterRole
- Check "where it takes effect" is RoleBinding or ClusterRoleBinding

## A critical understanding table

### Role
- Defines permissions within a namespace

### ClusterRole
- Defines cluster-wide permissions or reusable permission templates

### RoleBinding
- Binds permissions in a specific namespace
- Can bind to Role or ClusterRole

### ClusterRoleBinding
- Binds ClusterRole across the entire cluster

This understanding table is extremely important.

## A complete basic example

Now integrate the ServiceAccount from the previous article.

### 1. Create ServiceAccount

    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: app-reader
      namespace: default

### 2. Create Role

    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      name: pod-reader
      namespace: default
    rules:
      - apiGroups: [""]
        resources: ["pods"]
        verbs: ["get", "list", "watch"]

### 3. Create RoleBinding

    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: bind-pod-reader
      namespace: default
    subjects:
      - kind: ServiceAccount
        name: app-reader
        namespace: default
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: Role
      name: pod-reader

### 4. Create a Pod using this ServiceAccount

    apiVersion: v1
    kind: Pod
    metadata:
      name: app-reader-pod
      namespace: default
    spec:
      serviceAccountName: app-reader
      containers:
        - name: nginx
          image: nginx:1.25

These four steps together form the most basic:

**Identity + Permissions + Binding + User**

Complete chain.

## Common usage scenarios

### 1. An application that only reads Pods in its own namespace

Use:

- ServiceAccount
- Role
- RoleBinding

### 2. A controller that needs to read Node information across the entire cluster

Use:

- ServiceAccount
- ClusterRole
- ClusterRoleBinding

### 3. A generic permission rule that needs to be reused across multiple namespaces

Use:

- ClusterRole
- Multiple RoleBindings

This is why ClusterRole isn't just "for the entire cluster"—it's also commonly used as a reusable template.

## Common misconceptions

### Misconception 1: RoleBinding can only bind to Role

Not true.  
RoleBinding can also bind to ClusterRole.

### Misconception 2: ClusterRole always means global effect

Not true.  
If bound via RoleBinding, it can also take effect only in a specific namespace.

### Misconception 3: Creating a Role means you already have permissions

Not true.  
Without Binding, the rules won't automatically be assigned to any subject.

### Misconception 4: A ServiceAccount has corresponding business permissions once created

Not true.  
ServiceAccount is just an identity—permissions require RBAC binding.

### Misconception 5: RBAC is only for humans

Not true.  
In application deployment scenarios, RBAC is more often used for Pods/ServiceAccounts.

## Understanding approach in production

In production environments, it's recommended to understand RBAC this way:

### Step 1: First determine who the subject is
For example:
- A specific ServiceAccount
- A specific controller
- A specific system component

### Step 2: Clearly define its responsibilities
For example:
- Read-only access to Pods
- Read ConfigMap
- Manage Lease
- Watch Node

### Step 3: Grant it only the necessary permissions
For example:
- Don't give delete if it's not needed
- Don't give Secret if it's not needed
- Don't give ClusterRoleBinding if cluster permissions aren't needed

This is what we'll discuss later:

**Principle of least privilege**

## How to view existing RBAC objects

View Role:

    kubectl get role -n default

View RoleBinding:

    kubectl get rolebinding -n default

View ClusterRole:

    kubectl get clusterrole

View ClusterRoleBinding:

    kubectl get clusterrolebinding

View detailed content: /think

kubectl describe role pod-reader -n default  
kubectl describe rolebinding bind-pod-reader -n default  
kubectl describe clusterrole node-reader  
kubectl describe clusterrolebinding bind-node-reader  

## Troubleshooting: Why is Pod Access to API Denied  

If your application encounters:  

- Forbidden  
- cannot list resource  
- cannot get resource  
- permission denied  

typically follow this order of checks.  

### Step 1: Confirm Which ServiceAccount the Pod is Using  

    kubectl get pod <pod名> -o yaml  

Check:  

    serviceAccountName  

### Step 2: Confirm That This ServiceAccount Exists  

    kubectl get sa -n <namespace>  

### Step 3: Confirm There Is a Corresponding Binding  

Check:  

    kubectl get rolebinding -n <namespace>  
    kubectl get clusterrolebinding  

### Step 4: Confirm Which Role / ClusterRole Is Bound  

Check:  

    roleRef  

### Step 5: Confirm That the Permission Rules Actually Include the Target Resource and Action  

For example, if the application reports it cannot list pods, check:  

- whether resources include `pods`  
- whether verbs include `list`  

This troubleshooting chain is very commonly used.  

## A Minimal Experiment Recommendation  

### Experiment Objective  

Understand how Role and RoleBinding grant a ServiceAccount read-only access to pods within a namespace.  

### Step 1: Create ServiceAccount  

    apiVersion: v1  
    kind: ServiceAccount  
    metadata:  
      name: demo-reader  
      namespace: default  

### Step 2: Create Role  

    apiVersion: rbac.authorization.k8s.io/v1  
    kind: Role  
    metadata:  
      name: demo-pod-reader  
      namespace: default  
    rules:  
      - apiGroups: [""]
        resources: ["pods"]  
        verbs: ["get", "list", "watch"]  

### Step 3: Create RoleBinding  

    apiVersion: rbac.authorization.k8s.io/v1  
    kind: RoleBinding  
    metadata:  
      name: demo-bind-reader  
      namespace: default  
    subjects:  
      - kind: ServiceAccount  
        name: demo-reader  
        namespace: default  
    roleRef:  
      apiGroup: rbac.authorization.k8s.io  
      kind: Role  
      name: demo-pod-reader  

### Step 4: Apply Resources  

    kubectl apply -f demo-sa.yaml  
    kubectl apply -f demo-role.yaml  
    kubectl apply -f demo-rolebinding.yaml  

### Step 5: View Objects  

    kubectl get sa -n default  
    kubectl get role -n default  
    kubectl get rolebinding -n default  
    kubectl describe role demo-pod-reader -n default  
    kubectl describe rolebinding demo-bind-reader -n default  

This experiment focuses on helping you connect the relationships between the four core objects.  

## Key Points Recap  

Remember these core points:  

1. `ServiceAccount` answers "who am I", while RBAC answers "what can I do"  
2. RBAC core is about controlling "who, which resources, and what actions"  
3. `Role` defines namespace-level permissions  
4. `ClusterRole` defines cluster-level permissions or generic permission templates  
5. `RoleBinding` binds permissions within a namespace  
6. `ClusterRoleBinding` binds permissions across the entire cluster  
7. `RoleBinding` can bind either Role or ClusterRole  
8. Without a Binding, even a complete Role won't automatically grant permissions  
9. The principle of least privilege is the core idea of RBAC practice  

## Common Command Quick Reference  

    kubectl get sa -n default  
    kubectl get role -n default  
    kubectl get rolebinding -n default  
    kubectl get clusterrole  
    kubectl get clusterrolebinding  
    kubectl describe role <role名> -n default  
    kubectl describe rolebinding <binding名> -n default  
    kubectl describe clusterrole <clusterrole名>  
    kubectl describe clusterrolebinding <clusterrolebinding名>  
    kubectl get pod <pod名> -o yaml  

## One-Sentence Summary  

RBAC fundamentally answers this core question: **What resources in Kubernetes can this identity perform which operations on.**  

## Tags  
#Kubernetes #RBAC #Role #ClusterRole #RoleBinding #ClusterRoleBinding #ServiceAccount #MinimumPermissions

## Operational Extension Understanding
- Kubernetes Official Documentation: Using RBAC Authorization  
  https://kubernetes.io/docs/reference/access-authn-authz/rbac/
- Kubernetes Official Documentation: Service Accounts  
  https://kubernetes.io/docs/concepts/security/service-accounts/
- Kubernetes Official Documentation: Authorization Overview  
  https://kubernetes.io/docs/reference/access-authn-authz/authorization/

## Next Day Plan
- Study [[03-Least Privilege Practice - Assign Read-Only Permissions to Business Pods]]
- Combine ServiceAccount with RBAC to create a real-world authorization case
- Understand why "being able to run" and "reasonable permission design" are two separate concepts