# 05 - Summary of Application Identity and Permission Control Phase

## Document Description
- Document Purpose: Summary of Kubernetes application identity and permission control phase
- Applicable Phases: 07 - Application Deployment / 09 - Application Identity and Permission Control
- Learning Objectives:
  - Understand how ServiceAccount, RBAC, minimal permissions, and high-risk accounts work together
  - Comprehend the role of "identity" and "permission" in Kubernetes
  - Develop basic methodologies for designing Pod identities and permissions
  - Foster a security mindset that emphasizes clear application identities, need-based permissions, and default constraints
  - Lay a solid foundation for subsequent Helm deployments, middleware setups, and platform component installations

## What Is Learned in This Phase

On the surface, this phase covers:

- `ServiceAccount`
- `Role`
- `ClusterRole`
- `RoleBinding`
- `ClusterRoleBinding`
- Principles of minimal permissions
- Risks associated with high-risk accounts

However, at a higher level, this phase aims to address two core questions:

### 1. Who Is the Pod in the Cluster?
This is about the **identity issue**.

### 2. What Can This Identity Do?
This is about the **permission issue**.

The essence of this phase is not to memorize object names but to establish the following main thread:

**After a Pod starts running, what identity does it use to access APIs in Kubernetes, and what specific actions are allowed?**

## Why Follow This Phase After Application Deployment

Previous steps, such as scheduling, Deployment management, Service configuration, and release update rollback, dealt with:

- How Pods start running
- Where Pods are located
- How applications are released and updated

However, once an application is running, if it needs to interact with the Kubernetes control plane, another set of issues arises:

- Does it have an identity?
- Can it access APIs?
- Which resources can it read?
- Could incorrect permission design pose security risks?

Therefore, the logical sequence is:

1. First, make sure the Pod starts running.
2. Then, determine who the Pod is in the cluster.
3. Next, figure out what actions it can perform.
4. Finally, ensure that these permissions are appropriate, secure, and minimized.

This is the significance of the application identity and permission control phase.

## Part 1: ServiceAccount Solves Identity Issues

### What Problem Does It Solve?

`ServiceAccount` addresses the question of:

**Who is the Pod in the Kubernetes cluster?**

It provides an identity for Pods, controllers, Agents, and workloads, but not for human login purposes.

It is important to distinguish between two types of identities:

- **User Identity**: Humans use `kubectl`, platforms, or kubeconfig to manage the cluster.
- **Pod Identity**: Running workloads use this identity to access Kubernetes APIs.

`ServiceAccount` belongs to the second category.

## Understanding the Essence of ServiceAccount

You can think of it as:

**The system account/application account for Pods in the cluster.**

Its main functions are:

- Providing an identity identifier
- Establishing an authentication context when accessing APIs
- Serving as a carrier for RBAC permission models

In other words, `ServiceAccount` is only responsible for answering the question "Who am I?" not "What can I do?" This is one of the most fundamental concepts in this phase.

## Why Should We Pay Special Attention to the Default ServiceAccount?

Each Namespace has a default `serviceAccount` by default.

If a Pod does not specify a `serviceAccountName`, it will usually use the default one.

This means that:

- Not specifying an account does not mean there is no identity.
- However, using the default account may lead to security risks.

From production and security perspectives, it is not recommended to rely on the default account for many business scenarios, as this can result in:

- Unclear identity boundaries
- Difficulties in auditing
- Potential widespread impacts if misconfigured
- Challenges in later permission adjustments

Therefore, the general rule for ServiceAccount usage is:

**If identities can be separated independently, do not rely on the default account for long periods.**

## Part 2: RBAC Solves Permission Issues

If `ServiceAccount` is considered the "ID badge," then RBAC is the "access control system."

RBAC addresses the question of:

**What actions can this identity perform on which resources?**

Its core principles can be summarized in three key aspects:

- **Who**: The entity performing the action.
- **Resources**: The targets of the action.
- **Actions**: The specific operations allowed.

## Understanding the Four Core RBAC Objects

The four most important RBAC objects in this phase are:

- `Role`
- `ClusterRole`
- `RoleBinding`
- `ClusterRoleBinding`

You can categorize them into two types:

### Type 1: Define Permission Rules
- `Role`
- `ClusterRole`

### TypeTherefore, in business scenarios, cluster-level binding must be done with great caution.

## The Complete Methodology You Should Establish at This Stage

By now, you should have summarized identity and access control into the following methodology.

### Step 1: Determine Whether the Pod Needs Access to the Kubernetes API
Not all businesses require this.

### Step 2: If Needed, Create a Separate ServiceAccount for It
Do not blindly reuse `default`.

### Step 3: Define Only the Necessary Permissions for It
Grant only the necessary permissions on specific resources and verbs.

### Step 4: Limit Permissions to the Minimum Scope Possible
If it can be resolved within a Namespace, do not escalate it to the cluster level.

### Step 5: Avoid Direct Connections Between High-Permission Accounts and Business Pods
In particular, avoid reusing default accounts and excessive ClusterRoleBindings.

These five steps form the basic methodology for applying identity and access control.

## You Can Condense This Stage into a Mind Map

### 1. Who Is the Pod in the Cluster?
- `ServiceAccount`

### 2. What Can It Do?
- `Role`
- `ClusterRole`

### 3. How Are Permissions Granted to It?
- `RoleBinding`
- `ClusterRoleBinding`

### 4. How Should Permissions Be Designed?
- Minimum permissions
- Default constraints
- Independent identities
- On-demand authorization

### 5. What Are the Consequences of Incorrect Permission Design?
- Increased attack surface
- Higher risk of misoperations
- Increased exposure of Secrets
- Difficulties in auditing and governance

This is the core framework for this stage.

## How to Think When Conducting Actual Troubleshooting

If you encounter issues with an application accessing the Kubernetes API or are preparing to review a permission design, it is recommended to follow this sequence of thought:

### 1. Which ServiceAccount Is Used by This Pod?
Check:

    `kubectl get pod <pod-name> -o yaml`

Focus on:

    `serviceAccountName`

### 2. Is This ServiceAccount an Independent Identity?
Or is it reusing `default`?

### 3. What Role/ClusterRole Is It Bound To?
Check:

    `kubectl get rolebinding -n <namespace>`
    `kubectl get clusterrolebinding`

### 4. Are These Permissions Sufficient?
Consider:

- Resource scope
- Verb scope
- Involvement of Secrets
- Whether it Escalates to the Cluster Level

### 5. Are There Any Obvious Over-Permission Configurations?
For example:

- A business Pod is bound to `cluster-admin`
- The default account has excessive permissions
- A Role includes many unrelated resources
- Read-only access is granted but write permission is also provided

This approach is very useful for subsequent permission inspections and security reviews.

## Why This Stage Is Important for Further Learning

No matter what you learn later on—whether it's:

- Helm deployment
- Deploying Kubernetes on middleware
- Monitoring component deployment
- Log component deployment
- Operators/Controllers
- Platform components
- Troubleshooting production-grade applications—you will inevitably face this practical issue:

**What identity does this component use in the cluster, and are its permissions reasonable?**

Many issues such as components failing to start, encountering "Forbidden" errors, running abnormally, or posing significant security risks are directly related to the content of this stage.

Therefore, this part is not just theoretical but forms the foundation for a large number of practical applications later on.

## What You Should Really Remember at This Stage Is Not YAML, But Three Sentences

### The First Sentence
**ServiceAccounts handle identities, while RBAC manages permissions.**

### The Second Sentence
**Minimum permission means giving exactly what is needed, not less.**

### The Third Sentence
**High-permission accounts pose not only security risks but also long-term governance challenges.**

Once you truly grasp these three points, most permission design issues will become clear.

## Summary of Stage Knowledge

### 1. ServiceAccount
- Identifies the Pod in the cluster
- Solves the "who am I" problem

### 2. Role
- Namespace-level permission rules

### 3. ClusterRole
- Cluster-level permission rules or universal permission templates

### 4. RoleBinding
- Grants permissions within a specific namespace

### 5. ClusterRoleBinding
- Grants permissions throughout the entire cluster

### 6. Minimum Permission
- Grants only the minimum required permissions
- No pre-granting or unnecessary expansion

### 7. Risks of High Permissions
- Wider impact in case of breaches
- Greater damage from misoperations
- Increased exposure of Secrets
- Difficulties in auditing and control

## Key Points to Review at This Stage

You need to firmly understand the following core concepts:

1. `ServiceAccounts` resolve the identity of Pods in the cluster.
2. RBAC manages the operational permissions associated with identities.
3. `Role