# 03 - Least Privilege Practice: Assigning Read-Only Permissions to Business Pods

## Documentation Overview
- Document Location: Introduction to Kubernetes Application Identity and Permission Control Practices
- Applicable Phase: 07 - Application Deployment / 09 - Application Identity and Permission Control
- Learning Objectives:
  - Understand why business pods should follow the least privilege principle.
  - Master the basic practices of combining ServiceAccounts, Roles, and RoleBindings for authorization.
  - Be able to configure "read-only permissions" for a business pod.
  - Comprehend why excessive permissions should not be granted initially.
  - Develop a fundamental security mindset of "clear identities, restrained permissions, and授权 on demand."

## Establish an Intuition First

As we learned in the previous article:

- `ServiceAccounts` address the question of "who I am."
- `RBAC` addresses the question of "what I can do."

However, in actual production, what truly matters is not whether you know how to configure RBAC, but whether:

**the permissions you grant are exactly what is needed.**

Because once Kubernetes permissions are set too broadly, the risks can increase significantly.

For example:

- A regular business pod may only need read access to Pod information within its own namespace.
- If you directly grant it delete, update, and create permissions, it could be catastrophic.
- In even more severe cases, granting it high-level permissions across the entire cluster could allow an attacker to exploit it to manipulate other resources.

Therefore, the focus of this article is not on "how to grant permissions arbitrarily," but on:

**how to grant only the minimum amount of permission necessary for the pod to perform its tasks.**

## What is the Least Privilege Principle?

The least privilege principle can be simply understood as:

**Grant only the lowest level of permission required for a subject to complete its current task, and no more.**

For example:

- If read access is sufficient, do not grant write access.
- If a pod only needs to view Pod information, do not grant access to Secrets.
- If it only needs access within its own namespace, do not grant cluster-wide permissions.
- If `get`, `list`, and `watch` are enough, do not include `create`, `update`, or `delete`.

This is one of the most important security principles in Kubernetes permission control.

## Why Do Business Pods Need Least Privilege Even More?

Many people mistakenly think that:

- "Business pods aren’t system components, so granting them some permissions won’t cause major issues."
- "It’s better to start with broader permissions to avoid future errors."
- "Using the default settings or directly assigning high-level permissions is the easiest way."

However, this is extremely risky.

Business pods have the following characteristics:

- They are numerous.
- Their configurations change frequently.
- They have complex dependencies.
- They are more likely to contain security vulnerabilities.
- They are more vulnerable to attacks.

If you grant them unnecessary high-level permissions, the risks can be much greater than you anticipate.

For example:

### 1. Granting write access when only read access is needed
An attacker could use this to modify resources, delete Pods, or alter configurations.

### 2. Granting Secret read access when only ConfigMap access is required
This could directly expose secrets, passwords, and certificates.

### 3. Granting ClusterRoleBinding instead of just namespace-level permissions
This could cause issues that affect the entire cluster, rather than just a single business pod.

Therefore, for business pods, least privilege is not an optional practice; it is a fundamental security baseline.

## A Typical Business Scenario

Let’s consider a common scenario:

There is a business pod that needs to read the list of Pods in its current namespace during runtime for purposes such as:

- Assisting with service discovery.
- Printing diagnostic information.
- Checking the status of Pods in its environment.
- Performing simple monitoring or health checks.

For this requirement, it only needs:

- Read access to Pods.
- Read-only access.
- Access limited to its own namespace.

The most appropriate authorization would be:

- An independent ServiceAccount.
- A Role that grants read-only access to Pods within the namespace.
- A RoleBinding that binds this Role to the ServiceAccount.

This constitutes the standard least privilege authorization chain.

## Practical Objectives

In this article, our goal is to do exactly this:

**Grant a business pod "read-only Pod permissions," rather than broader ones.**

You will see that although this configuration is not complicated, it embodies a correct approach:

- Separate identities.
- Constrained permissions.
- Namespace isolation.
- Minimized actions.

## Step 1: Create an Independent ServiceAccount

Do not use the default `default` ServiceAccount; instead, create an independent identity for this business pod.

    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: app-pod-reader
      namespace: default

This means that:

- In the `default`namespace: default

### Role

    apiVersion: rbacauthorization.k8s.io/v1
    kind: Role
    metadata:
      name: read-pods-only
      namespace: default
    rules:
      - apiGroups: [""]
        resources: ["pods"]
        verbs: ["get", "list", "watch"]

### RoleBinding

    apiVersion: rbacauthorization.k8s.io/v1
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

This is a very standard example of "minimum read-only permissions for business pods."

## Why this design is good

It has several advantages:

### 1. Independent identities
It does not reuse the `default` namespace.

### 2. Limited scope of permissions
It only applies within the `default` namespace.

### 3. Narrow range of resources
It affects only `pods`.

### 4. Restricted actions
It allows only `get`, `list`, and `watch` operations.

### 5. Easy for auditing and maintenance
Whenever you see `app-pod-reader`, you know what it is used for.

This is a typical configuration that balances "security" and "maintainability."

## Going further: If a business does not need to access the API at all

It's important to understand this:

Not all business pods require RBAC permissions.  
And certainly, not all need to access the Kubernetes API.

If a business pod:

- Only provides a web service within its container
- Does not read cluster resources
- Does not call the API Server
- Does not rely on Kubernetes control plane capabilities

then it ideally should not:

- Need additional RBAC permissions
- Require automatic token mounting

For example:

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

This actually grants fewer permissions than just giving a "read-only Role."

Therefore, it's crucial to develop a higher level of security awareness:

**Minimum permission means not just "giving less," but also asking first: Is it really necessary at all?**

## When read-only pod permissions are needed

Common scenarios include:

### 1. Applications that need to monitor the status of pods in their own namespace
For example, for lightweight service discovery, diagnostics, or observability support.

### 2. Some platform-side agents that require reading business pod information
Such as for data collection, auditing, or monitoring purposes.

### 3. Automation scripts in operations that only need to observe without making any changes
For instance, checking if a pod is ready or if it exists at all.

### 4. Debugging tools or temporary operational utilities
These scenarios are particularly suitable for "read-only" access without the need to write data.

## When these permissions might still be considered excessive

Although this is about "read-only," it's important to remember:

**Read-only does not mean no risk at all.**

For example, reading pod information could reveal:

- The pod name
- Its namespace
- The node it is on
- Image details
- Tag information
- Status
- Environmental configuration clues

All of this information can be valuable to attackers.

Therefore, when determining minimum permissions, further questions should be asked:

- Do all pods need to be read?
- Or just a specific namespace?
- Or only certain types of resources?
- Is even watching not necessary?

Permission optimization is an ongoing process, not something done once and for all.

## Common mistakes

### Mistake 1: Directly assigning a ClusterRoleBinding
If a business only needs to read pods in its own namespace, using a cluster-level binding is excessive.

### Mistake 2: Assigning high-privilege ClusterRoles
Using roles like `edit` or `admin` for business pods is usually overkill.

### Mistake 3: Inclusion of secrets without necessity
If secrets are not required, they should not be included in the permission settings.

### Mistake 4: Granting write permissions initially and then revising them
This is a common practice of "granting more permissions initially to save effort," but these often remain unchanged later on.

### Mistake 5: Reusing the `defaultContainers:
        - name: nginx
          image: nginx:1.25

### Step 5: Apply and Verify

    kubectl apply -f demo-sa.yaml
    kubectl apply -f demo-role.yaml
    kubectl apply -f demo-rolebinding.yaml
    kubectl apply -f demo-pod.yaml

    kubectl get sa -n default
    kubectl get role -n default
    kubectl get rolebinding -n default
    kubectl describe pod demo-readonly-pod

The focus of this experiment is not to demonstrate technical prowess, but to thoroughly implement the principle of "independent identities + minimal permissions."

## Key Points to Remember

You should keep the following core concepts in mind:

1. The principle of minimal permissions is a fundamental security approach in Kubernetes authorization.
2. Business pods should not be granted broader permissions just for convenience.
3. The standard process involves using `ServiceAccount`, `Role`, `RoleBinding`, and `Pod` together.
4. If read-only access is sufficient, do not grant write permissions.
5. Limit permissions to a specific namespace instead of the entire cluster.
6. Grant only necessary permissions to pods and avoid including unrelated resources like ConfigMap, Secret, or Deployment.
7. If a service does not require access to the Kubernetes API, consider not granting authorization or even disabling automatic token mounting.
8. Security issues can arise not only from insufficient but also from excessive permissions.

## Common Commands for Quick Reference

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

## In One Sentence

The essence of minimal permission practice is to ensure that **only the minimum necessary permissions are granted to business pods to perform their tasks.**

## Tags

#Kubernetes #RBAC #ServiceAccount #Minimal Permissions #Application Security #Role #RoleBinding #Pod Permissions

## Additional Resources for Operations and Maintenance

- Kubernetes Official Documentation: Using RBAC Authorization  
  https://kubernetes.io/docs/reference/access-authn-authz/rbac/
- Kubernetes Official Documentation: Service Accounts  
  https://kubernetes.io/docs/concepts/security/service-accounts/
- Kubernetes Official Documentation: Good Practices for Kubernetes Secrets  
  https://kubernetes.io/docs/concepts/security/secrets-good-practices/

## Next Steps

- Study [[04-Why Direct Use of High-Level Permissions Is Not Recommended]].
- Understand the actual security risks associated with high-level ServiceAccounts, default account reuse, and overly broad RBAC bindings.
- Develop a awareness that incorrect permission design can also pose potential production hazards.