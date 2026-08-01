# 04 - Why Not to Directly Use High-Privilege Accounts

## Document Notes
- Document Location: Kubernetes Application Identity and Permission Control Security Awareness
- Applicable Stage: 07 - Application Deployment / 09 - Application Identity and Permission Control
- Learning Objectives:
  - Understand why it's not recommended to directly use high-privilege accounts for business Pods
  - Understand the risks of high-privilege ServiceAccounts, default account reuse, and overly broad RBAC bindings
  - Be able to identify common "for convenience, give broad permissions" errors
  - Establish awareness that permission design errors are production incident risks
  - Lay the foundation for forming minimal permissions, security baselines, and audit habits

## First Establish an Intuition

In the previous section we discussed:

**You shouldTry. only assign the minimum permissions required for the current task to business Pods.**

So this section answers the opposite question:

**Why is it not recommended to directly use high-privilege accounts?**

Many people have seen similar practices in actual work:

- Give broader permissions first to avoid application errors
- Let business Pods reuse the default account directly
- Bind a ready-made high-privilege ClusterRole directly
- Some components that are only read-only are given write permissions by default
- For convenience, directly use `cluster-admin`

These practices may seem convenient in the short term, but will almost certainly create risks in the long run.

Because in Kubernetes, the risks of high-privilege accounts are not "theoretically possible", but:

**Once an issue occurs, the impact will be rapidly amplified.**

## What Constitutes a High-Privilege Account

The "high-privilege account" mentioned here doesn't necessarily refer to a special name; it fundamentally means:

**An identity with permissions exceeding actual needs.**

Common manifestations include:

- Having `create / update / patch / delete` write permissions but the business only needs read access
- Being able to access `secrets` but the business doesn't need it
- Being able to operate the entire Namespace or even the entire cluster
- Gaining cluster-wide permissions through `ClusterRoleBinding`
- Directly binding `cluster-admin`, `admin`, `edit`, etc., overly broad roles
- Multiple businesses sharing a highly privileged ServiceAccount

So "high-privilege" isn't an absolute value, but rather relative to actual needs:

**The portion exceeding needs is the risk.**

## Why People Often Accidentally Use High-Privilege Accounts

This is very common in production environments, and the reasons are usually not complex:

### 1. For Quick Troubleshooting

When an application reports `Forbidden`, the easiest approach is:

- Directly increase permissions
- Even give high-privilege permissions all at once

### 2. To Save Configuration Time

Fine-grained Role, RoleBinding, and ServiceAccount configurations require writing multiple YAML files.  
Many people find this tedious.

### 3. Poor Understanding of RBAC

Not knowing exactly which resources and verbs to assign, so they "go all in" with a broad approach.

### 4. Following Historical Practices

Others have configured it this way before, so it's copied again.

### 5. Mistaking "Being Able to Run" as "Reasonable"

If the application starts up, it's assumed the configuration is correct.  
But "being able to run" and "being secure and reasonable" are two different things.

## Where Exactly Is the Risk of High-Privilege Accounts?

This is the core of this section.

You can understand the risks of high-privilege accounts from three levels:

- Security risks
- Risk of accidental operations
- Operation governance risks

## First Type of Risk: If a Pod is Compromised, Its Permissions Will Be Exploited

This is the most realistic risk.

Pods run business code, third-party dependencies, images, and runtime components.  
None of these are absolutely secure.

If a Pod is exploited, for example:

- The application has a vulnerability
- The container is compromised to get a shell
- The image contains malicious code
- A dependency library has an RCE issue
- Configuration exposure leads to intrusion

The attacker gains more than just "a container"—they may also get:

**The permissions of the ServiceAccount associated with this Pod.**

If this account has high privileges, the attacker could continue:

- Reading more cluster resources
- Accessing Secrets
- Viewing node information
- Deleting or modifying business resources
- Affecting other Namespaces laterally
- Even taking control of the entire cluster

So the fundamental risk of high-privilege accounts is:

**Turning "a single Pod being compromised" into a "cluster-level security incident."**

## Second Type of Risk: Accidental Operations Expand the Impact Scope

Not all issues come from attackers; many result from accidental operations by people themselves.

For example, a business Pod that only performs read queries is given write permissions.  
A code bug, script bug, or abnormal logic might lead to:

- Accidentally deleting a Pod
- Accidentally modifying a ConfigMap
- Accidentally updating a Lease
- Accidentally overwriting certain objects
- Accidentally triggering cascading issues

If the permissions were only read-only, these errors would never occur.  
But with larger permissions, bugs can turn into incidents.

In other words:

**The larger the permissions, the greater the destructive radius of errors.**

## Third Type of Risk: Secret Exposure Scope Is Unnecessarily Expanded

In Kubernetes, `Secret` is a very sensitive resource.

If an account is granted read access to Secrets, it may access:

- Database passwords
- API Tokens
- TLS Certificates
- Private keys
- Internal system authentication information
- Third-party platform access keys

Many business Pods actually don't need to access Kubernetes Secret API,  
They only need to get their own configuration through environment variables or mounting.

If you "conveniently" also grant Secret read access, it's equivalent to:

**Expanding the access scope of credentials that shouldn't be exposed.**

This is a very typical high-risk error.

## Fourth Type of Risk: Default Account Reuse Causes Ambiguous Boundaries

In many environments, business Pods default to using:

    default

This ServiceAccount.

If permissions are directly bound to `default`, a very dangerous situation arises:

- Many Pods in the same Namespace share the same identity
- It's hard to distinguish which Pods truly need these permissions
- Granting permissions to one place benefits many Pods
- A problem in one place makes troubleshooting and root cause analysis difficult
- Subsequent permission consolidation becomes very challenging

The most typical manifestation of this issue is:

**Originally intending to grant permissions to one business, but ending up with all Pods in the entire Namespace gaining them.**

From a governance perspective, directly expanding permissions on `default` is a poor practice.

## Fifth Type of Risk: ClusterRoleBinding Amplifies Issues Across the Entire Cluster

This is a more severe situation than granting permissions within a Namespace.

If a business ServiceAccount is bound to:

- `ClusterRoleBinding`
- Broad-ranging `ClusterRole`
- Even `cluster-admin`

The risks are no longer limited to a single Namespace, but may expand to:

- All cluster Pods
- All cluster Namespaces
- Nodes
- Secrets
- CRD / CR
- Cluster-level configurations

This means that if an ordinary business Pod has an issue, it could affect the entire Kubernetes platform, not just a single business.

So you can understand it this way:

- `RoleBinding` risks usually have limited scope
- `ClusterRoleBinding` risks are inherently larger
- For business Pods, cluster-level bindings must be handled with extreme caution

## Sixth Type of Risk: Auditing and Permission Consolidation Become Very Difficult

Permission design isn't just about whether the applications can run today, but also about whether they can be managed in the future.

If many businesses: /think

- Reuse default accounts  
- Use shared high-privilege accounts  
- No identity splitting by business  
- Role and Binding chaos  

You will encounter these issues later:  

### 1. Don't know who truly needs these permissions  
Because many businesses share the same authorization set.  

### 2. Dare not revoke permissions  
Fear of revoking permissions might break other businesses.  

### 3. Audit difficulties  
It's hard to quickly identify who used this identity to perform an action when issues occur.  

### 4. Permission debt accumulates continuously  
Initially, more permissions were given for convenience, but later no one dares to adjust.  

The problem of high-privilege accounts is not just a security issue, but also a long-term governance issue.  

## A typical error case  

Assume a business Pod only needs to read Pods in its own namespace.  

The correct approach should be:  

- Independent ServiceAccount  
- Role  
- Only give `pods`  
- Only give `get/list/watch`  
- RoleBinding bound to the current namespace  

But the wrong approach might be:  

- Directly use `default`  
- Directly bind `edit`  
- Even bind to `cluster-admin`  

This approach may be "fastest" in the short term, but risks are immediately maximized.  

Because `edit` or higher permissions usually mean:  

- Can modify more resources  
- Can create and delete objects  
- Permissions scope far exceeds business actual needs  

This configuration essentially means:  

**Trade convenience for future risks.**  

## Why "give more first, then restrict later" usually can't be reversed  

This is a very real production experience.  

Many teams initially say:  

- Let it run first  
- Restrict permissions later  

But reality is usually:  

- No one has time to organize later  
- More applications, more complex dependencies  
- No one dares to act, fearing errors from incorrect restrictions  
- Eventually, this oversized permission becomes solidified  

So you need to establish a realistic understanding:  

**Expanding permissions is easy, but converging them is hard.**  

This is why you should design with minimal permissions from the start.  

## High-privilege accounts are also a risk for business teams  

Not just platform teams, business teams themselves are also affected.  

If a business account has too much permission, then:  

- Business code errors cause larger damage  
- Application security vulnerabilities have more severe consequences  
- Higher risk of going live  
- Post-incident troubleshooting and responsibility attribution become more complex  

So minimal permissions are not "platforms making extra trouble for business," but:  

**Protecting the platform while also protecting the business itself.**  

## Which roles usually require special caution  

You don't need to memorize all built-in roles now, but you should have a basic judgment:  

These roles, once given to business Pods, usually require high vigilance:  

- `cluster-admin`  
- `admin`  
- `edit`  
- Roles with `secrets` read permissions  
- Roles with `create/update/delete` wide-range write permissions  
- Any ClusterRoleBinding at arbitrary cluster level  

Especially:  

**Business Pods rarely truly need cluster-admin.**  

If you see such bindings in your business YAML, your first reaction should not be "convenient," but "dangerous."  

## When might relatively high permissions be needed  

High permissions aren't always forbidden.  
Some system components, platform controllers, and operations control planes indeed need relatively large permissions, for example:  

- Operator  
- Cluster monitoring and scheduling components  
- Node management components  
- Backup and recovery platform  
- Platform-level controllers  

Even for these components that need high permissions, you should ensure:  

- Clear permission boundaries  
- Dedicated ServiceAccount  
- Clear purpose  
- Proper review and audit  
- Not shared with regular business  

In other words:  

**High permissions aren't absolutely forbidden, but must have clear justification and be used under control.**  

## Why default ServiceAccount needs special caution  

This point is worth emphasizing separately.  

If you bind high permissions to `default`, what problems will arise?  

### 1. Strong default propagation  
Many Pods that don't explicitly write `serviceAccountName` will inherit it.  

### 2. Easy to accidentally grant permissions to a large number of Pods  
You think you're granting permissions to a specific application, but other Pods in the same namespace also get them.  

### 3. Poor auditability  
Logs are all `default`, making it hard to quickly identify which business it is.  

### 4. Difficult to revoke permissions  
Because many Pods depend on this default identity, it's hard to split later.  

So a safer principle is:  

**Do not directly bind high permissions to default ServiceAccount.**  

## A better alternative approach  

If an application really needs permissions, don't directly "increase default" permissions. Instead, follow this order:  

### Step 1: Confirm what exact permissions the application needs  
Check the error message:  

- Can't get  
- Can't list  
- Can't watch  
- Can't create  
- Can't update  

Also confirm which resource it is.  

### Step 2: Create an independent ServiceAccount for the application  
Don't reuse public accounts.  

### Step 3: Only add the specific missing permissions  
Only add necessary resources and necessary verbs.  

### Step 4: Limit to the smallest scope  
Use namespace-level if possible, avoid cluster-level.  

### Step 5: Retain the minimal set after verification  
Don't expand unnecessarily.  

This process, though slower than "directly bind a large role," is the correct approach.  

## Re-understanding from the attack surface perspective  

If you view Kubernetes as a platform, RBAC is like the internal permission boundary of the platform.  

High-privilege accounts are equivalent to:  

- Having more access cards  
- Entering more rooms  
- Being able to operate more devices  
- Seeing more sensitive information  

Business Pods inherently have a larger exposure surface than control plane components because they:  

- Provide external services  
- Are more vulnerable to input attacks  
- Have more third-party dependencies  
- Have more frequent releases  

Giving high-privilege accounts to business Pods is like placing a larger key in a higher exposure area.  

This is why security baselines usually emphasize:  

**Business workloads should avoid holding Kubernetes API permissions beyond actual needs.**  

## How to determine if an account has excessive permissions  

You can quickly self-check with these questions.  

### 1. Does this Pod really need to access the Kubernetes API?  
If not, it shouldn't even have read permissions.  

### 2. Does it really need write permissions?  
If it's just observational, read-only is usually sufficient.  

### 3. Does it really need Secret permissions?  
Most businesses don't need direct access to Kubernetes Secret API.  

### 4. Does it really need cluster-level permissions?  
If it only acts within the same namespace, it shouldn't use ClusterRoleBinding.  

### 5. Why can't it use an independent ServiceAccount?  
If the answer is just "convenience," it's usually not a valid reason.  

If you can't answer any of these questions, you should suspect the permissions might be excessive.  

## What to look at when investigating and reviewing the current state  

### Check ServiceAccounts in the namespace  

    kubectl get sa -n default  

### Check RoleBinding  

    kubectl get rolebinding -n default  

### Check ClusterRoleBinding  

    kubectl get clusterrolebinding  

### Check specific binding details

kubectl describe rolebinding <binding名> -n default  
kubectl describe clusterrolebinding <binding名>  

### Viewing Which ServiceAccount a Pod Uses  

    kubectl get pod <pod名> -o yaml  

Focus on:  

    serviceAccountName  

If you discover:  

- Many services are using `default`  
- A particular service is bound to a cluster-level high-privilege role  
- A particular account can read secrets but the service doesn't need it  
- Many roles have excessive verbs  

This indicates that the permission design requires governance.  

## A Simple Negative Example  

The following example isn't meant to be copied, but to help you identify issues.  

    apiVersion: rbac.authorization.k8s.io/v1  
    kind: ClusterRoleBinding  
    metadata:  
      name: business-app-admin  
    subjects:  
      - kind: ServiceAccount  
        name: default  
        namespace: default  
    roleRef:  
      apiGroup: rbac.authorization.k8s.io  
      kind: ClusterRole  
      name: cluster-admin  

This configuration has serious issues:  

- Uses `default`  
- Directly binds to `cluster-admin`  
- Applies to the entire cluster  
- Many Pods in the default namespace might be affected  

Such configurations in a business environment are typically considered high-risk.  

## A More Reasonable Approach  

When encountering permission issues, don't first think:  

**How to quickly grant permissions.**  

Instead, first think:  

**What exactly is missing, and how much minimum permission is needed?**  

These two mindsets are vastly different.  

The former continuously creates permission debt.  
The latter is how to build a secure, maintainable, and auditable system.  

## Key Points to Remember  

You need to remember these core points:  

1. The risk of high-privilege accounts fundamentally lies in "exposing permissions beyond real needs to workloads"  
2. If a business Pod is compromised, an attacker may inherit its ServiceAccount permissions for lateral exploitation  
3. Excessive permissions not only pose security risks but also amplify the risk of accidental operations  
4. `default` ServiceAccount reuse leads to blurred boundaries, expanded impact scope, and audit difficulties  
5. `ClusterRoleBinding` expands issues from namespaces to the entire cluster  
6. "Granting more first, then restricting later" often fails to be reversed in practice  
7. Business Pods should typically not directly use `cluster-admin`, `admin`, `edit`, etc., high-privilege roles  
8. Permission design errors are themselves production incident risks  
9. The correct approach isn't "grant first," but "authorize precisely by need, resource, action, and scope"  

## Common Command Quick Reference  

    kubectl get sa -n default  
    kubectl get rolebinding -n default  
    kubectl get clusterrolebinding  
    kubectl describe rolebinding <binding名> -n default  
    kubectl describe clusterrolebinding <binding名>  
    kubectl get pod <pod名> -o yaml  
    kubectl describe sa <sa名> -n default  

## One-Sentence Summary  

The fundamental reason not to directly use high-privilege accounts is: **If a business Pod has issues, excessive Kubernetes permissions will rapidly escalate a localized problem into a more severe platform-level risk.**  

## Tags  
#Kubernetes #RBAC #ServiceAccount #MinimumPermissions #ApplicationSecurity #ClusterRoleBinding #AccountNumber #SecurityBaseline  

## Operations Extension Understanding  
- Kubernetes Official Documentation: Using RBAC Authorization  
  https://kubernetes.io/docs/reference/access-authn-authz/rbac/  
- Kubernetes Official Documentation: Service Accounts  
  https://kubernetes.io/docs/concepts/security/service-accounts/  
- Kubernetes Official Documentation: Authorization Overview  
  https://kubernetes.io/docs/reference/access-authn-authz/authorization/  
- Kubernetes Official Documentation: Good Practices for Kubernetes Secrets  
  https://kubernetes.io/docs/concepts/security/secrets-good-practices/  

## Next Day Plan  
- Study [[05-Application Identity and Access Control Phase Summary]]  
- Connect ServiceAccount, RBAC, minimal permissions, and high-privilege risks into a complete understanding  
- Establish a foundational security mindset of "clear application identity, permission by need, and default convergence"