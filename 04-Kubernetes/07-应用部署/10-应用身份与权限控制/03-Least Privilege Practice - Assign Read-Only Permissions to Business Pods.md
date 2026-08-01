# 03-Practicing Minimal Permissions: Assigning Read-Only Permissions to Business Pods

## Document Notes
- Document Location: Kubernetes Application Identity and Permission Control Practice Guide
- Applicable Stage: 07-Application Deployment / 09-Application Identity and Permission Control
- Learning Objectives:
  - Understand why business Pods should follow the principle of minimal permissions
  - Master the basic practice of combining ServiceAccount, Role, and RoleBinding for authorization
  - Be able to configure "read-only permissions" for a business Pod
  - Understand why large permissions should not be granted from the start
  - Establish a basic security awareness of "clear identity, restrained permissions, and needs-based authorization"

## First Establish an Intuition

In the previous article, we already knew:

- `ServiceAccount` solves "who I am"
- `RBAC` solves "what I can do"

But in actual production, what's truly important isn't "whether you can configure RBAC", but:

**Are the permissions you grant exactly what's needed?**

Because Kubernetes permissions that are too broad can rapidly escalate risks.

For example:

- A regular business Pod only needs to read Pod information in its own namespace
- But you directly grant it delete, update, and create permissions
- In more severe cases, you even grant it high-level permissions across the entire cluster

Once this Pod is compromised, its code exploited, or its container taken over, attackers could use its identity to perform lateral operations across cluster resources.

So the core of this article isn't "how to casually grant permissions", but:

**How to grant exactly the minimal permissions needed to complete the work.**

## What Is the Principle of Minimal Permissions

The principle of minimal permissions can be simply understood as:

**Grant only the lowest level of permissions required for the current task, no more.**

For example:

- If only read is needed, don't grant write
- If only Pod viewing is needed, don't grant Secret access
- If only namespace-level access is needed, don't grant cluster-wide access
- If only `get/list/watch` is needed, don't grant `create/update/delete`

This is one of the most important security principles in Kubernetes permission control.

## Why Business Pods Need Minimal Permissions More

Many people mistakenly believe:

- "Business Pods aren't system components, so giving them some permissions isn't a big deal"
- "Give them more permissions first to avoid errors later"
- "Using default or directly granting high permissions is the easiest way"

This is actually very dangerous.

Because business Pods have the following characteristics:

- Large numbers
- Frequent changes
- Complex dependencies
- More prone to code vulnerabilities
- More likely to be exploited by attackers

Once you grant them unnecessary high permissions, the risks can be much greater than you imagine.

For example:

### 1. Only needs to read Pods, but granted write permissions
Attackers could use this to modify resources, delete Pods, or change configurations.

### 2. Only needs to read ConfigMaps, but granted Secret reading permissions
This could directly expose credentials, passwords, and certificates.

### 3. Only needs namespace-level permissions, but granted ClusterRoleBinding
This could escalate the issue from "a single business" to "affecting the entire cluster."

For business Pods, minimal permissions aren't an option—they're the basic security baseline.

## A Typical Business Scenario

Let's set up a very common scenario:

A business Pod needs to read the list of Pods in its current namespace during runtime, for:

- Service discovery assistance
- Printing diagnostic information
- Reading the status of Pods in its own environment
- Simple observation or health check logic

Under this requirement, it actually only needs:

- Read Pod
- Read-only
- Limited to the current namespace

The most reasonable authorization should be:

- A dedicated ServiceAccount
- A Role for read-only Pods within the namespace
- A RoleBinding that binds this Role to the ServiceAccount

This is the standard minimal permission authorization chain.

## Practice Goal

What we need to do in this article is:

**Assign "read-only Pod permissions" to a business Pod, not larger permissions.**

You'll see that although this configuration isn't complex, it embodies a complete set of correct approaches:

- Independent identity
- Permission convergence
- Namespace isolation
- Minimal actions

## Step 1: Create an Independent ServiceAccount

Don't use the default `default` ServiceAccount, but instead create an independent identity for this business.

    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: app-pod-reader
      namespace: default

This indicates:

- In the `default` namespace
- Create an identity specifically for the business Pod
- Named `app-pod-reader`

Apply:

    kubectl apply -f sa-app-pod-reader.yaml

Check:

    kubectl get sa -n default
    kubectl describe sa app-pod-reader -n default

## Why Create an Independent ServiceAccount First

Because this is the first step in minimal permissions:

**Split the identity first.**

If multiple businesses share `default`, you'll face many problems later:

- Unclear auditing
- Unclear permission boundaries
- A single adjustment affects many Pods
- Difficult to determine "who should have what permissions"

So the first step is always:

**Clarify identity and avoid using default accounts.**

## Step 2: Create a Read-Only Pod Role

Define a Role that allows only reading Pods in the current namespace.

    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      name: read-pods-only
      namespace: default
    rules:
      - apiGroups: [""]
        resources: ["pods"]
        verbs: ["get", "list", "watch"]

This Role means:

- Scope: `default` namespace
- Target resource: `pods`
- Allowed actions:
  - `get`
  - `list`
  - `watch`

These three actions are a typical combination for read-only operations.

## Why Use get, list, and watch

These verbs are particularly common and are often used by many read-only components.

### get
Read a single object, such as viewing details of a Pod.

### list
List objects, such as viewing all Pods in the current namespace.

### watch
Continuously monitor resource changes, such as watching Pod status changes.

If a component is for observation, reading, or discovery purposes, these three actions are usually sufficient.

## Why Not Add create, update, delete

Because the current business need is "read Pod information."

It doesn't need:

- Creating Pods
- Updating Pods
- Deleting Pods
- Patching Pods

So these verbs shouldn't be added.

This is exactly the core of minimal permissions:

**Grant only what is clearly needed now, and don't grant permissions in advance for hypothetical future needs.**

## Why Are secrets, configmaps, and deployments Not Added Here

Because the current requirements do not mention these resources.

This is also a very common misconception:

Many people writing Roles tend to add many resources at once, for example:

- pods
- deployments
- configmaps
- secrets
- services

The usual reasoning is "it might be needed in the future".

But from a security perspective, this approach is substandard.  
Because every additional resource surface introduces another risk surface.

Especially:

**Secrets should almost never be granted just because it's convenient.**

## Step 3: Bind the Role to the ServiceAccount

Now that we have identity and permission rules, the final step remains: binding.

    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: bind-read-pods-only
      namespace: default
    subjects:
      - kind: ServiceAccount
        name: app-pod-reader
        namespace: default
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: Role
      name: read-pods-only

The meaning of this RoleBinding is:

- In the `default` namespace
- Assign the `read-pods-only` permissions
- To the `app-pod-reader` ServiceAccount

Only then does the authorization chain truly establish.

## Step 4: Let the Pod Use This ServiceAccount

Now create a business Pod and specify that it uses this independent identity.

    apiVersion: v1
    kind: Pod
    metadata:
      name: business-app
      namespace: default
    spec:
      serviceAccountName: app-pod-reader
      containers:
        - name: nginx
          image: nginx:1.25

Apply:

    kubectl apply -f pod-business-app.yaml

Check:

    kubectl describe pod business-app

Focus on confirming:

    Service Account:  app-pod-reader

This indicates that the Pod is no longer using the default identity, but instead the restricted identity we designed earlier.

## So What's the Minimal Privilege Chain Here

You can compress the entire process into four steps:

### 1. Create an Independent Identity
`ServiceAccount`

### 2. Define Minimal Permissions
`Role`

### 3. Bind Identity and Permissions
`RoleBinding`

### 4. Let the Business Pod Use This Identity
`serviceAccountName`

This is the most fundamental and correct Kubernetes business authorization pattern.

## Complete Example Summary

Below are the four objects connected together for your reference.

### ServiceAccount

    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: app-pod-reader
      namespace: default

### Role

    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      name: read-pods-only
      namespace: default
    rules:
      - apiGroups: [""]
        resources: ["pods"]
        verbs: ["get", "list", "watch"]

### RoleBinding

    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: bind-read-pods-only
      namespace: default
    subjects:
      - kind: ServiceAccount
        name: app-pod-reader
        namespace: default
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: Role
      name: read-pods-only

### Pod

    apiVersion: v1
    kind: Pod
    metadata:
      name: business-app
      namespace: default
    spec:
      serviceAccountName: app-pod-reader
      containers:
        - name: nginx
          image: nginx:1.25

This is a very standard "business Pod minimal read-only permissions" case.

## Why This Design Is Good

It's good for the following reasons.

### 1. Independent Identity
Not reusing `default`.

### 2. Narrow Permission Scope
Only effective in the `default` namespace.

### 3. Narrow Resource Scope
Only applies to `pods`.

### 4. Narrow Action Scope
Only includes `get/list/watch`.

### 5. Easy to Audit and Maintain Later
In the future, seeing `app-pod-reader` will immediately tell you what it does.

This is a typical configuration that balances "security" and "maintainability".

## Further Consideration: If the Business Doesn't Need to Access the API

Here's an important cognitive point to add.

Not all business Pods need RBAC permissions.  
Even less so do they need to access the Kubernetes API.

If a business Pod:

- Simply provides a web service in the container
- Does not read cluster resources
- Does not call the API Server
- Does not depend on Kubernetes control plane capabilities

Ideally, it should even:

- Not need additional RBAC
- Not require automatic token mounting

For example: /think

apiVersion: v1
kind: Pod
metadata:
  name: simple-web
  namespace: default
spec:
  automountServiceAccountToken: false
  containers:
    - name: nginx
      image: nginx:1.25

This is actually a smaller permission than "granting a read-only Role".

So you need to establish a higher level of security awareness:

**Minimum permissions aren't just "giving a little less", but first ask: do you really need to give it at all.**

## When do you need read-only Pod permissions

Common scenarios include:

### 1. The application needs to perceive the status of Pods in the current namespace
For example, for lightweight service discovery, diagnostics, or observability assistance.

### 2. Some platform-side Agents need to read business Pod information
For example, for monitoring, auditing, or inspection.

### 3. SomeTransport automation logic only needs to observe, not change
For example, checking if a Pod is ready or exists.

### 4. Debugging or temporaryTransport tools
This scenario is especially suitable for "read-only, no write".

## When this permission still counts as large

Although this is a "read-only permission", you also need to know:

**Read-only doesn't mean there's no risk.**

For example, reading Pods might mean you can see:

- Pod name
-belong to Namespace
- Node it resides on
- Image information
- Label information
- Status information
- Environmental structure clues

These information could still be valuable to attackers.

So minimum permissions need to continue asking:

- Do you really need to read all Pods?
- Or just read a specific Namespace?
- Or just read certain resources?
- Even whether watching is needed?

Permission convergence is an ongoing thinking process, not a one-time action.

## Common mistakes

### Mistake 1: Directly granting ClusterRoleBinding
The business only needs to read Pods in its own namespace, but directly uses cluster-level binding.

This greatly expands the permission scope.

### Mistake 2: Directly binding existing high-permission ClusterRole
For example, taking the shortcut to bind `edit`, `admin` type roles.

This is usually excessive in business Pod scenarios.

### Mistake 3: Adding secrets together by default
If not specified in the requirements, don't add them.

### Mistake 4: Giving write permissions first and talking about it later
This is the most typical "to save time, just open it up" habit, and later people usually won't revoke it.

### Mistake 5: Reusing default for all Pods
Short-term convenience, long-term chaos and danger.

## More reasonable permission designThinking. in production

You can think about it in the following order.

### Step 1: Does this Pod really need to access Kubernetes API
If not, don't authorize, even disable token automatic mounting.

### Step 2: If it does, what resources does it need to read
For example, read-only Pods, read-only ConfigMaps, instead of "reading many things".

### Step 3: Does it need to read or write
If it's read-only, only give read verbs.

### Step 4: Where is the permission scope
If it can be limited to a specific namespace, don't expand it to the entire cluster.

### Step 5: Give dedicated identity, don't reuse public accounts
This step is very important for subsequent auditing and convergence.

## TroubleshootingThinking.: Is the permission too large or wrong

In actual work, permission issues aren't only about "insufficient permissions", but also "excessive permissions".

It's recommended to check from two directions.

### Direction 1: Insufficient permissions
For example, the application reports errors:

- forbidden
- cannot list pods
- cannot get resource

This indicates the Role/Binding might not be configured properly.

### Direction 2: Excessive permissions
For example, when you review a business YAML, you find:

- Using `default`
- Bound ClusterRoleBinding
- Verbs with create/update/delete
- Resources with secrets
- Clearly only needs to read current namespace, but bound cluster role

This is a typical case of "excessive permission design".

From a security engineering perspective, the latter is also a problem.

## How to view this authorization object

View ServiceAccount:

    kubectl get sa -n default
    kubectl describe sa app-pod-reader -n default

View Role:

    kubectl get role -n default
    kubectl describe role read-pods-only -n default

View RoleBinding:

    kubectl get rolebinding -n default
    kubectl describe rolebinding bind-read-pods-only -n default

View Pod:

    kubectl get pod business-app -o yaml
    kubectl describe pod business-app

## A minimal experiment suggestion

### Experiment goal

Configure a business Pod with minimal permissions to only read Pods in the current namespace.

### Step 1: Create ServiceAccount

    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: demo-readonly-sa
      namespace: default

### Step 2: Create Role

    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      name: demo-pod-readonly
      namespace: default
    rules:
      - apiGroups: [""]
        resources: ["pods"]
        verbs: ["get", "list", "watch"]

### Step 3: Create RoleBinding

    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: demo-pod-readonly-binding
      namespace: default
    subjects:
      - kind: ServiceAccount
        name: demo-readonly-sa
        namespace: default
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: Role
      name: demo-pod-readonly

### Step 4: Create business Pod

apiVersion: v1
kind: Pod
metadata:
  name: demo-readonly-pod
  namespace: default
spec:
  serviceAccountName: demo-readonly-sa
  containers:
    - name: nginx
      image: nginx:1.25

### Step 5: Apply and Check

    kubectl apply -f demo-sa.yaml
    kubectl apply -f demo-role.yaml
    kubectl apply -f demo-rolebinding.yaml
    kubectl apply -f demo-pod.yaml

    kubectl get sa -n default
    kubectl get role -n default
    kubectl get rolebinding -n default
    kubectl describe pod demo-readonly-pod

The focus of this experiment isn't about showcasing functionality, but to actually run through the "independent identity + minimal permissions" approach in practice.

## Key Points to Remember

You need to remember the following core concepts:

1. The principle of least privilege is the fundamental security principle for Kubernetes application authorization  
2. Business Pods shouldn't be granted larger permissions just for convenience  
3. The standard workflow is: `ServiceAccount + Role + RoleBinding + Pod`  
4. Only grant read access if writing isn't needed  
5. Only grant access to specific namespaces instead of cluster-wide  
6. Only grant access to Pods, not ConfigMap, Secret, or Deployment by default  
7. If the business doesn't need to access Kubernetes API, consider denying authorization or even disabling token auto-mounting  
8. Security issues aren't only about "insufficient permissions", but also include "excessive permissions"

## Common Command Quick Reference

    kubectl get sa -n default
    kubectl describe sa app-pod-reader -n default
    kubectl get role -n default
    kubectl describe role read-pods-only -n default
    kubectl get rolebinding -n default
    kubectl describe rolebinding bind-read-pods-only -n default
    kubectl get pod business-app -o yaml
    kubectl describe pod business-app
    kubectl apply -f sa.yaml
    kubectl apply -f role.yaml
    kubectl apply -f rolebinding.yaml
    kubectl apply -f pod.yaml

## One-Sentence Summary

The core of least privilege practice isn't "whether to grant permissions", but: **Grant only the minimal permissions required for the business Pod to complete its current task, no more.**

## Tags
#Kubernetes #RBAC #ServiceAccount #MinimumPermissions #ApplicationSecurity #Role #RoleBinding #PodPermissions

## Operations Extension Understanding
- Kubernetes Official Documentation: Using RBAC Authorization  
  https://kubernetes.io/docs/reference/access-authn-authz/rbac/
- Kubernetes Official Documentation: Service Accounts  
  https://kubernetes.io/docs/concepts/security/service-accounts/
- Kubernetes Official Documentation: Good practices for Kubernetes Secrets  
  https://kubernetes.io/docs/concepts/security/secrets-good-practices/

## Next Day Plan
- Study [[04-Why Not to Recommend Direct Use of High Privilege Accounts]]
- Understand the actual security risks brought by high-privilege ServiceAccount, default account reuse, and overly broad RBAC bindings
- Establish awareness that "permission design errors also constitute production incident risks"