# 04-Why It's Not Recommended to Use High-Level Permission Accounts Directly

## Document Description
- Document Focus: Understanding the Security Implications of Identity and Access Control in Kubernetes Applications
- Applicable Phases: 07-Application Deployment / 09-Application Identity and Access Control
- Learning Objectives:
  - Understand why it's not advisable for business Pods to directly use high-level permission accounts.
  - Recognize the risks associated with reusing high-permission ServiceAccounts, default accounts, or overly broad RBAC bindings.
  - Identify common practices of granting excessive permissions for convenience.
  - Develop the awareness that incorrect permission design can pose production safety hazards.
  - Lay the foundation for implementing minimal permission principles, security baselines, and audit practices.

## Establishing an Intuitive Understanding First

In the previous section, we discussed:

**We should strive to assign only the minimum necessary permissions to business Pods to complete their tasks.**

So, this section aims to address the question:

**Why is it not recommended to use high-level permission accounts directly?**

Many people have encountered similar practices in their work:

- Granting broader permissions initially to avoid application errors.
- Allowing business Pods to reuse default accounts.
- Directly assigning a pre-existing high-permission ClusterRole.
- Giving read-only components write access out of convenience.
- Using `cluster-admin` without careful consideration.

These approaches may seem convenient in the short term but often lead to long-term problems.

In Kubernetes, the risks associated with high-level permission accounts are not theoretical; rather, they are:

**Once an issue arises, their impact can be rapidly exacerbated.**

## What Are High-Level Permission Accounts?

Here, "high-level permission accounts" do not necessarily refer to specific account names but rather mean:

**Accounts that possess more permissions than are actually required.**

Common examples include:

- Accounts with `create / update / patch / delete` and other write permissions when only read access is needed.
- Accounts that can access `secrets` when no such need exists.
- Accounts with the ability to manage entire namespaces or even the entire cluster.
- Accounts that obtain cluster-wide permissions through `ClusterRoleBinding`.
- Direct assignments of overly broad roles like `cluster-admin`, `admin`, or `edit`.
- Multiple business services sharing a single highly privileged ServiceAccount.

Therefore, "high-level" is relative; what constitutes high permission depends on the actual needs of the service:

**Any excess permissions pose a risk.**

## Why Do So Many People Accidentally Use High-Level Permission Accounts?

This issue is quite common in production environments, and the reasons are usually straightforward:

### 1. For Quick Troubleshooting

When an application reports a `Forbidden` error, the simplest solution is often to grant broader permissions.

### 2. To Save Configuration Time

Creating separate Roles, RoleBindings, and ServiceAccounts requires additional YAML configuration, which many people find cumbersome.

### 3. Lack of Understanding of RBAC

Not knowing exactly which resources or verbs require哪些 permissions, some people simply assign everything without careful consideration.

### 4. Following Historical Practices

If others have done it this way in the past, it is often continued without question.

### 5. Mistaking Functionality for Rationality

Once an application starts working, it is assumed that the configuration is correct. However, functionality does not equate to security and rationality.

## What Are the Dangers of High-Level Permission Accounts?

This section covers the core risks associated with such accounts:

- **Security Risks**: If a Pod is compromised, the high-level permissions associated with that account can be exploited to access and manipulate other resources in the cluster.
- **Risk of Accidental Misoperations**: Misconfiguring permissions can lead to unexpected deletions, modifications, or other errors that can cause significant damage.
- **Operational and Governance Risks**: Overly broad permission settings can make it difficult to manage and audit the system, leading to potential compliance issues.

## The First Type of Risk: Permission Inheritance in Case of a Pod Compromise

This is the most immediate risk. Pods often contain sensitive code, dependencies, images, and runtime components, all of which are not inherently secure. If a Pod is compromised, an attacker may gain access to:

- Additional cluster resources.
- Secrets containing sensitive information.
- Node configurations.
- Business data.

With high-level permissions, the attacker can potentially cause widespread damage, affecting other pods, namespaces, or even the entire cluster.

## The Second Type of Risk: Expanded Impact Due to Misoperations

Many problems arise not from external attacks but from internal misconfigurations. For example, if a Pod that is supposed to perform only read operations is granted write access, it may accidentally delete important data or modify critical configurations.

## The Third Type of Risk: Unnecessary Exposure of Secrets

`Secrets` in Kubernetes contain highly sensitive information such as passwords, API tokens, and TLS certificates. If an account with read access to secrets is misconfigured, theseMany Pods that do not explicitly specify a `serviceAccountName` will inherit it.

### 2. It is easy to inadvertently affect a large number of Pods
You might think you are granting permissions to a specific application, but in reality, other Pods in the same namespace also obtain those permissions.

### 3. Poor auditability
The logs are filled with `default`, making it difficult to quickly identify which specific business it relates to.

### 4. Difficulty in restricting permissions
Since many Pods rely on this default identity, it becomes very hard to adjust them later on.

Therefore, a more prudent approach is:

**Do not directly assign high permissions to the default ServiceAccount.**

## A better alternative approach

If an application truly lacks certain permissions, instead of simply increasing the privileges for `default`, follow this sequence:

### Step 1: Determine exactly what permissions are missing
Check the error messages to see if it's related to:
- inability to get
- inability to list
- inability to watch
- inability to create
- inability to update

Also, identify which specific resource is affected.

### Step 2: Create a separate ServiceAccount for the application
Do not reuse a public account.

### Step 3: Grant only the missing permissions
Provide only the necessary resources and verbs.

### Step 4: Limit the scope to the minimum required
If it's sufficient within the namespace, avoid granting cluster-level permissions.

### Step 5: Retain only the necessary set of permissions after verification
Do not expand the privileges unnecessarily.

Although this process is a bit slower than directly assigning broader roles, it is the correct way to proceed.

## Understanding from an attack perspective

If you consider Kubernetes as a platform, RBAC acts like the internal permission boundaries within that platform.

High-privilege accounts are equivalent to:

- having more access cards
- entering more rooms
- being able to operate more devices
- having access to more sensitive information

Business Pods naturally have a larger exposure surface than control components because they:

- provide services externally
- are more vulnerable to external attacks
- rely on many third-party components
- frequently undergo changes

Therefore, granting high permissions to business Pods is like placing larger keys in areas with higher exposure risks.

This is why security best practices often emphasize that:

**Business workloads should avoid holding more Kubernetes API permissions than they actually need.**

## How to determine if an account has been over-privileged

You can quickly self-assess using the following questions:

### 1. Does this Pod really need access to the Kubernetes API?
If not, it shouldn't even have read permissions.

### 2. Does it really need write permissions?
For observation purposes, read-only usually suffices.

### 3. Does it really need Secret permissions?
Most businesses do not need direct access to Kubernetes Secrets APIs.

### 4. Does it really need cluster-level permissions?
If it only operates within its own namespace, a ClusterRoleBinding is unnecessary.

### 5. Why can't it use a separate ServiceAccount?
If the reason is just "for convenience," it is usually not a valid justification.

If you cannot answer any of these questions, it may indicate that the permissions have been over-allocated.

## What to check when investigating and reviewing the current situation

### Check ServiceAccounts in the namespace

    kubectl get sa -n default

### Check RoleBindings

    kubectl get rolebinding -n default

### Check ClusterRoleBindings

    kubectl get clusterrolebinding

### View specific binding details

    kubectl describe rolebinding <binding_name> -n default
    kubectl describe clusterrolebinding <binding_name>

### Check which ServiceAccount a Pod is using

    kubectl get pod <pod_name> -o yaml

Pay attention to:

    serviceAccountName

If you find that:

- Many businesses are using `default`
- A certain business has been granted cluster-level high permissions
- An account has read access to secrets, but the business does not need it
- There are too many role verbs assigned, then it indicates that the permission design needs improvement.

## A simple negative example

The following example is not meant to be copied, but rather to help you identify issues:

    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: business-app-admin
    subjects:
      - kind: ServiceAccount
        name: default
        namespace: default
    roleRef:
      apiGroup: rbacauthorization.k8s.io
      kind: ClusterRole
      name: cluster-admin

This configuration has serious problems:

- It uses `default`.
- It directly binds to `cluster-admin`.
- Its scope covers the entire cluster.
- Many Pods in the default namespace may be affected.

Such configurations should be regarded as high-risk items in a production environment.

## A more rational way of thinking

When encountering permission issues, instead of first considering:

**How to grant more permissions as