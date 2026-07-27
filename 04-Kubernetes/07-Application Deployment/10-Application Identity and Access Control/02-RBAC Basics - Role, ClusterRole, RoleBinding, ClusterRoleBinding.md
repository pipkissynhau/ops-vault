# 02-RBAC Basics: Role, ClusterRole, RoleBinding, ClusterRoleBinding

## Document Description
- Document Focus: Basics of Kubernetes Application Identity and Permission Control
- Applicable Phases: 07-Application Deployment / 09-Application Identity and Permission Control
- Learning Objectives:
  - Understand that "identity" and "permission" are distinct concepts in Kubernetes.
  - Comprehend how RBAC controls who can perform what actions.
  - Master the basic functions of `Role`, `ClusterRole`, `RoleBinding`, and `ClusterRoleBinding`.
  - Be able to interpret basic RBAC YAML configurations.
  - Lay a foundation for subsequent studies on minimal permission practices.

## Establishing an Initial Understanding

In the previous section, we learned about `ServiceAccount`.

That topic addressed:

**Who the Pod is within the cluster.**

However, having just an identity is not enough.  
In Kubernetes, other critical questions include:

- Whether it can access Pods.
- Whether it can access ConfigMap objects.
- Whether it can create Jobs.
- Whether it can delete resources.
- Whether it can read Secrets.
- Whether it has control over the entire cluster.

At this point, a set of "authorization rules" is needed to answer:

**What specific actions are allowed for this identity.**

This mechanism is RBAC.

## What is RBAC

RBAC stands for:

**Role-Based Access Control**

In Chinese, it is commonly referred to as:

**Based-Role Access Control**

It is the most widely used authorization model in Kubernetes and serves to control:

**Who can perform what actions on which resources.**

You can think of it this way:

- `ServiceAccount` determines "who you are."
- RBAC determines "what you can do."

These two concepts must be clearly distinguished.

## Why Does Kubernetes Need RBAC

Kubernetes is a platform with numerous resources and a high volume of operations.

There are many types of objects in a cluster:

- Pods
- Deployments
- Services
- ConfigMaps
- Secrets
- Namespaces
- Nodes
- Jobs
- Ingresses
- PersistentVolumes
- CustomResources

Different identities have vastly different needs regarding these resources. For example:

- A logging agent may only need access to Pods and Nodes.
- A controller might need to watch, list, and update certain resources.
- A backup component might require access to PVCs, PVs, and Pods.
- A business application might only need to read ConfigMaps within its own namespace.
- Some high-privilege components might even need management over all cluster resources.

Without permission control, two extreme scenarios would arise:

- Excessive permissions leading to high security risks.
- Insufficient permissions preventing applications from functioning properly.

Therefore, the goal of RBAC is to:

**Provide only the necessary permissions to complete tasks, nothing more.**

## The Core Issues of RBAC

You can think of RBAC as addressing these three questions:

### 1. Who
Who is attempting to access the resources?

For example:

- A certain `ServiceAccount`.
- A specific user.
- A user group.

### 2. What Resources
Which objects are being targeted for access?

For example:

- Pods.
- ConfigMaps.
- Secrets.
- Deployments.
- Nodes.

### 3. What Actions
What operations can be performed on these resources?

For example:

- Get.
- List.
- Watch.
- Create.
- Update.
- Patch.
- Delete.

In essence, RBAC is a combined authorization model based on:

**Subject + Resource + Action**

## Remember the Four Core Objects

The four most important objects in Kubernetes RBAC learning are:

- `Role`.
- `ClusterRole`.
- `RoleBinding`.
- `ClusterRoleBinding`.

You can categorize them into two dimensions:

### Permission Rule Objects
- `Role`.
- `ClusterRole`.

### Binding Objects
- `RoleBinding`.
- `ClusterRoleBinding`.

In other words:

- The first two define "what permissions are available."
- The latter two determine "to whom these permissions are granted."

## What is a Role

A `Role` defines:

**A set of permission rules within a specific namespace.**

It operates at the **namespace level**.

This means that a Role typically specifies:

- Within which namespace.
- Which resources can be accessed.
- What actions are allowed on those resources.

For example:

- Permission to read Pods in the `default` namespace.
- Permission to view ConfigMaps in a certain namespace.
- Permission to update Leases in a particular namespace.

You can think of a Role as:

**A permission checklist for a specific namespace.**

## What is a ClusterRole

A `ClusterRole` defines:

**A set of permission rules at the cluster level.**

Its main difference from a Role lies in its broader scope of application.

Common use cases include:

- Accessing cluster-level resources such as Nodes, PersistentVolumes, and### Step 1: Create a ServiceAccount
To address the question "Who am I?"

### Step 2: Create a Role
To define "What I can do."

### Step 3: Create a RoleBinding
To grant "These permissions to me."

This is the most fundamental authorization model in Kubernetes applications.

## Third Basic Example: ClusterRole

Let's look at an example of a ClusterRole.

Suppose a component needs to read information about all nodes in the cluster:

    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: node-reader
    rules:
      - apiGroups: [""]
        resources: ["nodes"]
        verbs: ["get", "list", "watch"]

This ClusterRole indicates that:

- It allows reading Node resources.
- This is a cluster-level resource access scenario.

Since `nodes` are not resources within a regular namespace but rather have a more global scope, ClusterRole is usually used in such cases.

## Fourth Basic Example: ClusterRoleBinding

If you want to bind this `node-reader` to a certain ServiceAccount and make it effective across the entire cluster, you can do so as follows:

    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: bind-node-reader
    subjects:
      - kind: ServiceAccount
        name: node-agent
        namespace: kube-system
    roleRef:
      apiGroup: rbacauthorization.k8s.io
      kind: ClusterRole
      name: node-reader

This YAML means that:

- It binds the `node-reader` ClusterRole to the `node-agent` in the `kube-system` namespace.
- This binding takes effect throughout the entire cluster.

This approach is commonly used for:

- Node monitoring components.
- Platform Agents.
- Cluster-level system components.

## RoleBinding Can Also Bind a ClusterRole

This is an important point that can easily cause confusion.

`RoleBinding` can not only bind `Roles` but also `ClusterRoles`.

### What Does This Mean?

It means that:

**I want to reuse a globally defined set of permission rules, but I only want them to be effective within a specific namespace.**

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
      apiGroup: rbacauthorization.k8s.io
      kind: ClusterRole
      name: view

This means that:

- A `ClusterRole` is used as the permission template.
- However, these permissions are only granted to `app-reader` within the `default` namespace.

Therefore, it's essential to distinguish between:

- What object the binding is applied to (Role or ClusterRole).
- Where the binding takes effect (RoleBinding or ClusterRoleBinding).

## A Very Important Understanding Chart

### Role
- Defines permissions within a namespace.

### ClusterRole
- Defines cluster-level permissions or reusable permission templates.

### RoleBinding
- Bonds permissions within a specific namespace.
- Can bind either Roles or ClusterRoles.

### ClusterRoleBinding
- Bonds a ClusterRole throughout the entire cluster.

This understanding chart is crucial.

## A Complete Basic Case Study

Now, let's integrate the ServiceAccount from the previous section into this process.

### 1. Create a ServiceAccount

    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: app-reader
      namespace: default

### 2. Create a Role

    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      name: pod-reader
      namespace: default
    rules:
      - apiGroups: [""]
        resources: ["pods"]
        verbs: ["get", "list", "watch"]

### 3. Create a RoleBinding

    apiVersion: rbacauthorization.k8s.io/v1
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

### 4. Create a Pod That Uses This ServiceAccount

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

These four steps together represent the most fundamental complete process:

**Identity + Permissions + Binding + User**

## Common Use Cases

### 1. A### Step 1: Determine which ServiceAccount the Pod is using

    Use the following command:
    `kubectl get pod <pod name> -o yaml`
    Check the `serviceAccountName` field.

### Step 2: Verify if this ServiceAccount exists

    Execute the following command:
    `kubectl get sa -n <namespace>`

### Step 3: Confirm whether there is a corresponding Binding

    Check the following commands:
    `kubectl get rolebinding -n <namespace>`
    `kubectl get clusterrolebinding`

### Step 4: Identify which Role/ClusterRole is bound

    Examine the `roleRef` field.

### Step 5: Verify if the permission rules actually include the target resources and actions

    For example, if an application reports an error indicating that it cannot list pods, check whether:
- The `resources` field includes `pods`
- The `verbs` field includes `list`

This series of troubleshooting steps is very commonly used.

## A Minimal Experiment Recommendation

### Experiment Objective

Understand how Roles and RoleBindings grant a ServiceAccount read-only permissions on Pods within a namespace.

### Step 1: Create a ServiceAccount

    Use the following command:
    ```yaml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: demo-reader
      namespace: default
    ```
    This creates a ServiceAccount named `demo-reader` in the `default` namespace.

### Step 2: Create a Role

    Use the following command:
    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      name: demo-pod-reader
      namespace: default
    rules:
      - apiGroups: []
        resources: ["pods"]
        verbs: ["get", "list", "watch"]
    ```
    This defines a Role named `demo-pod-reader` that allows operations on `pods` within the `default` namespace, such as getting, listing, and watching.

### Step 3: Create a RoleBinding

    Use the following command:
    ```yaml
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
      apiGroup: rbacauthorization.k8s.io
      kind: Role
      name: demo-pod-reader
    ```
    This binds the `demo-readers` ServiceAccount to the `demo-pod-reader` Role, granting it the necessary permissions.

### Step 4: Apply the Resources

    Use the following commands:
    ```bash
    kubectl apply -f demo-sa.yaml
    kubectl apply -f demo-role.yaml
    kubectl apply -f demo-rolebinding.yaml
    ```
    These commands apply the created ServiceAccount, Role, and RoleBinding resources to the cluster.

### Step 5: Verify the Objects

    Use the following commands:
    ```bash
    kubectl get sa -n default
    kubectl get role -n default
    kubectl get rolebinding -n default
    kubectl describe role demo-pod-reader -n default
    kubectl describe rolebinding demo-bind-reader -n default
    ```
    These commands display information about the created ServiceAccount, Role, and RoleBinding resources.

The focus of this experiment is to help you understand the relationship between these four core objects.

## Key Points to Remember

You need to keep in mind the following key points:
1. `ServiceAccounts` define who the user is.
2. RBAC determines what actions a user can perform on resources.
3. `Roles` define permissions within a namespace.
4. `ClusterRoles` define cluster-level or generic permission templates.
5. `RoleBindings` bind permissions to specific ServiceAccounts within a namespace.
6. `ClusterRoleBindings` bind permissions across the entire cluster.
7. `RoleBindings` can bind either Roles or ClusterRoles.
8. Without a binding, even the most complete Role will not grant any permissions.
9. The principle of least privilege is a fundamental concept in RBAC implementation.

## Common Commands for Quick Reference

    ```bash
    kubectl get sa -n default
    kubectl get role -n default
    kubectl get rolebinding -n default
    kubectl get clusterrole
    kubectl get clusterrolebinding
    kubectl describe role <role name> -n default
    kubectl describe rolebinding <binding name> -n default
    kubectl describe clusterrole <clusterrole name>
    kubectl describe clusterrolebinding <clusterrolebinding name>
    kubectl get pod <pod name> -o yaml
    ```
    These commands help you retrieve and display information about various Kubernetes objects.

## In One Sentence

RBAC essentially