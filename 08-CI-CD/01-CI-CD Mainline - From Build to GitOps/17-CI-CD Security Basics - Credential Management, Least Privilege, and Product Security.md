# 17-CI-CD Security Basics: Credential Management, Least Privilege, and Product Security

## Document Description

This article is the 17th note in the 08-CI-CD learning pathway.

In the previous 16 articles, we have successfully implemented the minimum release pipeline and gradually developed the following foundational capabilities:

- Manual release pipeline
- Image building and tag design
- Harbor repository
- GitLab CI and Jenkins Pipeline
- K8s deployment and rollback
- Multi-environment releases
- Helm value overrides and template reuse
- Basics of Runner / Agent executors

In this article, we will address a critical topic that is essential in any real-world production environment:

**CI/CD Security.**

When many people first start learning about pipelines, their focus tends to be on whether they can:

- Build the code
- Push it to the repository
- Deploy it to the environment

However, once they move into a formal production environment, they quickly encounter more critical issues:

- How should Harbor passwords be managed?
- How should kubeconfig be stored and accessed?
- Have Jenkins / GitLab CI been granted excessive permissions?
- Which accounts should be assigned to humans and which to machines?
- Is the content of the images reliable?
- Are any sensitive information exposed in the release process?

The goal of this article is not to provide a comprehensive security framework but to focus on three key aspects, based on the capabilities you have already established:

- Credential management
- Least privilege
- Product security

This article will continue to use the current learning environment and tools:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private repository
- `test` namespace
-已完成 build / push / deploy / rollback / Helm management for a minimal application

## Tags

#Kubernetes #CI-CD #Security #CredentialManagement #LeastPrivilege #Harbor #RobotAccounts #kubeconfig #ProductSecurity #PracticalNotes

## Learning Objectives

After completing this article, you should be able to:

1. Understand what CI/CD security primarily protects.
2. Grasp the basic principles of credential management.
3. Recognize why least privilege is a core security principle for pipelines.
4. Comprehend what product security entails and why images themselves are part of the security framework.
5. Identify the most vulnerable areas in your current pipeline setup.
6. Explain why being able to execute a process does not necessarily mean it can be done securely.

## Experimental Focus of This Article

This article is divided into four sections:

1. Identify the sensitive points in your current release pipeline.
2. Understand the basic principles of credential management.
3. Learn how least privilege is implemented in Harbor, K8s, and pipeline executors.
4. Recognize why images and related products are also part of the security context.

---

## Section 1: Identifying Natural Security Risks in Your Current Release Pipeline

You have already performed the following actions:

- `docker login Harbor`
- `docker push`
- `kubectl set image`
- `kubectl rollout status`
- Helm upgrade / rollback
- Understanding the structure of GitLab CI / Jenkins Pipeline
- Understanding Runner / Agent executors

Functionally, these actions are all correct.  
However, from a security perspective, this pipeline contains at least the following sensitive points:

### Sensitive Point 1: Harbor Login Credentials

You have executed the command `docker login your-Harbor-domain`.

This means that any subsequent operations involving image pushing—whether locally, through GitLab CI, or Jenkins—will require:

- Username
- Password
- Or Robot account credentials

### Sensitive Point 2: K8s Cluster Access Credentials

If you can execute commands like `kubectl -n test set image ...` or `kubectl -n test rollout status ...`, it indicates that your environment has some form of cluster access, such as:

- kubeconfig
- ServiceAccount
- Some kind of cluster identity mapping

### Sensitive Point 3: Pipeline Executor Permissions

Runner / Agent executors are not just used for simple tasks like echoing commands; they can also perform actions like:

- Pushing images to Harbor
- Deploying K8s resources
- Performing rollbacks
- Operating across different environments

Therefore, if these executors have excessive permissions, it can lead to serious security issues.

### Sensitive Point 4: The Images Themselves

Images are not ordinary files; they will eventually run within the cluster.  
If an image contains problematic content, such as:

- Debugging passwords
- Unnecessary tools
- Vulnerable base images
- Sensitive information

even if the release process is correct, it remains unsafe.

### Key Points to Understand at This Stage

CI/CD security is not just about preventing unauthorized access to the platform interface.  
It also involves ensuring that:

- Credentials are properly managed
- Permissions are limited to the minimum## Section 8: Understanding How to Apply Minimum Privileges to Runners / Agents

In the previous sections 14 and 15, you have already learned that:

- Runners / Agents are executors.
- In the Jenkins on K8s scenario, Agents are often Pods.

Now, let's add a layer of security understanding.

### It’s Not Better for an Executor to Have More Powers

If an executor possesses all of the following:

- The ability to push images via Harbor
- Administrative privileges over all K8s namespaces
- Permissions to delete resources
- Rights to deploy in all environments,

while it may seem convenient, it also poses significant risks.

### A More Reasonable Approach

Executor capabilities should be divided according to specific tasks. For example:

- A build executor: Should focus on Docker / Harbor operations.
- A deploy executor: Should specialize in using kubectl / Helm.
- A test executor: Should be dedicated to validation and scanning tasks.

### Key Concepts to Understand at This Stage

The core principle of executor security is also minimum privileges:

- Each type of task should use its specific executor.
- An executor should only have the necessary capabilities for its role.

---

## Section 9: What Is Product Security, and Why Are Images Considered Security Objects?

Many beginners tend to overlook this aspect.

You have been building and pushing images throughout the process, but images are not just ordinary files—they will eventually be deployed within a cluster for execution.

Therefore, images are classified as “products” and thus security objects.

### Why Should Images Be Included in Security Considerations?

Images may contain:

- High-risk vulnerabilities from older base images.
- Unnecessary debugging tools.
- Operations with root privileges that increase risks.
- Plain-text configurations.
- Private files.
- Malicious scripts or backdoors.

### Key Concepts to Understand at This Stage

The basic idea of “product security” is:

**Everything deployed into an environment must be trustworthy.**

CI/CD security not only protects the process but also the outcomes of that process.

---

## Section 10: The Most Important Product Security Awareness to Develop at This Stage

You don’t need to implement a complete enterprise-level security scanning platform right away, but you should develop four key awareness areas:

### Awareness 1: Avoid Using Unknown Versions of Base Images

The versions of images you have used so far, such as `nginx:1.27`, `busybox:1.35`, and `alpine:3.20`, are generally from reliable sources.

### Awareness 2: Do Not Include Sensitive Information in Images

Examples include:

- Plain-text passwords.
- Private keys.
- kubeconfig files.
- Token files.

### Awareness 3: Keep Images As Simple as Possible

More complex images tend to:

- Increase the attack surface.
- Make debugging easier but also increase risks.

### Awareness 4: Gradually Incorporate Vulnerability Scanning When Moving on with Harbor

This represents the natural progression from awareness to practical platform capabilities in product security.

---

## Section 11: Practical Exercise: Identifying Places Where Sensitive Information May Be Leaked in Your Current Experiment

You don’t need to add any new systems for this exercise; just conduct a self-check.

### Step 1: Check the Current Experiment Directory

Enter:

    cd ~/08-ci-cd/01-manual-release
    ls -la

Check if there are any files or directories containing:

- Harbor login credentials.
- kubeconfig files.
- Plain-text password scripts.
- Private configuration files.

### Step 2: Check the Helm Directory

Enter:

    cd ~/08-ci-cd/08-helm-lab/manual-web
    ls -la

Ensure that there are no hard-coded passwords in the `values` files and that kubeconfig, token files, etc., are not stored in this directory.

### Key Concepts to Understand at This Stage

Many security issues arise from simple mistakes, such as accidentally including sensitive information.

The most important habit to develop at this stage is:

**Do not store sensitive files in experiment directories.**

---

## Section 12: Practical Exercise: Check for Unsafe Practices in Image Configurations

Use your familiar Dockerfile to conduct this check.

### Step 1: Review the Dockerfile

Enter the application directory and view the Dockerfile:

    cd ~/08-ci-cd/01-manual-release
    cat Dockerfile

### Key Points to Check

1. Verify if any plain-text passwords are included in the Dockerfile.
2. Ensure that no unnecessary local files are being included in the image.
3. Confirm that the current image only contains the essential contents required for the experiment.

### Key Concepts to Understand at This Stage

Even though your current experimental images are simple, this exercise helps you develop an important intuition:

> When building an image, every file included must have a clear purpose.

---

## Section 13: Deriving a Minimum CI/CD Security Checklist from