# 17-CI-CD Security Fundamentals: Credential Management, Minimum Privileges, and Artifact Security

## Document Notes

This article is the 17th note in the 08-CI-CD learning path.

The previous 01-16 have already established a working minimal deployment pipeline and gradually filled in these foundational capabilities:

- Manual deployment pipeline
- Image building and tag design
- Harbor repository
- GitLab CI and Jenkins Pipeline
- K8s deployment and rollback
- Multi-environment deployment
- Helm values override and template reuse
- Runner / Agent executor basics

From this point onward, we begin addressing a criticalMain that is unavoidable in real environments:

**CI/CD Security.**

Many newcomers to pipelines often focus initially on:

- Can it build
- Can it push
- Can it deploy

But once entering production environments, they immediately face more critical issues:

- How to securely store Harbor credentials
- How to manage kubeconfig
- Whether Jenkins / GitLab CI has excessive permissions
- Which accounts should be for humans, which for machines
- Whether image content is trustworthy
- Whether sensitive information in the deployment pipeline is exposed

This article's goal is not to present a comprehensive security system, but to combine your already established pipeline to clearly explain:

- Credential management
- Minimum privileges
- Artifact security

This article continues to align with your current learningMain and experimental environment:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private repository
- `test` namespace
- Minimal application build / push / deploy / rollback / Helm management completed

## Tags

#Kubernetes #CI-CD #Clear. #CertificateManagement #MinimumPermissions #Harbor #RobotAccountNumber #kubeconfig #ProductSafety #I'llTakeYourNotes.

## Learning Objectives

After completing this article, you should understand:

1. What CI/CD security primarily protects
2. The fundamental principles of credential management
3. Why minimum privileges is the core security principle for pipelines
4. What artifact security is and why images themselves are part of the security boundary
5. How to identify the most vulnerable security points in your currentMain
6. How to explain why "being able to run" does not equal "being able to run securely"

## This Article's ExperimentalMain

This article is divided into 4 sections:

1. First identify sensitive points in the current deployment pipeline
2. Understand the basic principles of credential management
3. Understand how minimum privileges apply to Harbor, K8s, and pipeline executors
4. Understand why images and artifacts also belong to security objects

---

## Part 1: First examine the current deployment pipeline for naturally risky points

You have already completed these actions:

- `docker login Harbor`
- `docker push`
- `kubectl set image`
- `kubectl rollout status`
- Helm upgrade / rollback
- GitLab CI / Jenkins Pipeline structure understanding
- Runner / Agent executor understanding

From a functional perspective, these actions are fine.  
But from a security perspective, this chain has at least these sensitive points.

### Sensitive Point 1: Harbor Login Credentials

You executed:

    docker login your Harbor domain

This means that regardless of local, GitLab CI, or Jenkins, any action requiring pushing images will inevitably involve:

- Username
- Password
- Or Robot account credentials

### Sensitive Point 2: K8s Cluster Access Credentials

If you can execute:

    kubectl -n test set image ...
    kubectl -n test rollout status ...

It indicates the execution environment has some form of cluster access capability, such as:

- kubeconfig
- ServiceAccount
- Some form of cluster identity mapping

### Sensitive Point 3: Pipeline Executor Permissions

Runners / Agents aren't just running echo commands; they might actually be able to:

- Push to Harbor
- Deploy K8s
- Rollback
- Operate across different environments

So if the executor has excessive permissions, it becomes very dangerous.

### Sensitive Point 4: The Image Itself

Images aren't neutral files; they will eventually run in the cluster.  
If the image content itself is problematic, such as:

- Contains debug passwords
- Includes extra tools
- Base image has critical vulnerabilities
- The image contains sensitive information

Even if the deployment process is correct, it's still insecure.

### Understanding to Establish at This Stage

CI/CD security isn't just about "not letting others log into the platform page,"  
but rather that across the entire pipeline:

- Credentials
- Permissions
- Images
- Executors

All belong to security boundaries.

---

## Part 2: Understand the core of this article with one sentence

At this stage, you just need to remember one sentence:

> **The foundation of CI/CD security isn't about pursuing complexity first, but about achieving "no credential leaks, no privilege abuse, and not deploying untrusted images into the environment."**

This sentence is the main thread of this article.

---

## Part 3: Credential Management - Start with the most basic principles

So-called evidence., in the currentMain, the most typical include:

- Harbor username/password
- Harbor Robot account
- kubeconfig
- Secrets stored in future GitLab / Jenkins
- Future Helm repository authentication, Webhook Token, etc.

At this stage, remember the minimum 4 principles first.

### Principle 1: Do not write credentials in plain text into the repository

For example, you shouldn't directly write these contents into Git repository files:

- Harbor password
- kubeconfig
- Token
- Private key

#### Wrong Thinking Example

Putting things like the following directly into scripts or YAML:

    docker login harbor.example.com -u admin -p MyPassword123

Or directly submitting kubeconfig to the repository.

#### Correct Understanding

- Credentials are not part of business code
- Should not be propagated with the repository long-term
- Once committed, the propagation scope will be very wide

### Principle 2: Prioritize letting the platform manage credentials, rather than writing credentials into command history

When you eventually use GitLab CI / Jenkins, the more reasonable storage location for Harbor credentials should be:

- GitLab CI Variables / Secret
- Jenkins Credentials

Rather than:

- Writing into `.gitlab-ci.yml`
- Writing into Jenkinsfile
- Writing into shell script body

### Principle 3: Use machine accounts for different systems, avoid long-term shared human accounts

For example, pushing images to Harbor is better suited for:

- Robot account

Rather than long-term using an administrator's personal account.

### Principle 4: Credentials should be visible to as small a scope as possible

Only those who truly need it should have access to it.  
For example:

- The build process doesn't necessarily need K8s kubeconfig
- The deploy process doesn't necessarily need Harbor push permissions

### Understanding to Establish at This Stage

The core of credential management is not "hiding passwords very mysteriously,"  
but:

- Don't scatter
- Don't transmit randomly
- Don't let irrelevant people and systems know

---

## Part Four: Combining with Harbor, Understanding Why "Machine Accounts" Are More Secure

This section directly combines what you've already learned about Harbor.

You've already known that Harbor has two types of identities:

- Regular user account
- Robot account

### When is a Regular User Account More Suitable

- Log in to Harbor page
- Manually view images
- Manage projects and repositories
- Do permission configuration

### When is a Robot Account More Suitable

- GitLab CI push images
- Jenkins Pipeline push images
- Automated systems pull / push
- Machine-to-machine calls

### Why Robot Accounts Are More Suitable for Pipelines

Because pipelines are "system behavior", not "human manual login behavior".

If pipelines long-term use a human administrator account, there are several issues:

1. Usually excessive permissions
2. Not conducive to distinguishing responsibility boundaries
3. More difficult to manage when personnel change
4. Not conducive to password rotation

### Understanding to Establish at This Stage

As long as a capability is "long-term automated use", we should prioritize thinking:

- Does a machine account exist?
- Can we give it minimal permissions separately?

Rather than first using an administrator account as a makeshift solution.

---

## Part Five: Minimum Permissions, Why It Is the Core Principle of CI/CD Security

This section is very critical.

The meaning of minimum permissions is simple:

**A person, a system, or an executor should only have the minimal capabilities needed to complete the current task.**

### Why This Is Important

Because once a pipeline has excessive permissions, the impact when something goes wrong will be very significant.

For example:

- A process that should only push Harbor ends up being able to delete images
- A process that should only deploy to test environment ends up being able to modify production
- A user account that should only view a certain project ends up being able to view the entire repository

### Understanding to Establish at This Stage

CI/CD security is not first about "more complex tools",  
but first about:

**Do not give more permissions than needed.**

---

## Part Six: How to Understand Minimum Permissions on Harbor

Combined with the current main line, Harbor's minimum permissions at least manifest in the following directions.

### 1) Project Isolation

For example:

- Different business images are placed in different Harbor projects
- Different teams only see their own project's repositories

### 2) Account Boundaries

For example:

- Regular users only do management or viewing
- Robot accounts only do push / pull for certain projects

### 3) Do Not Default to Highest Permissions

If a pipeline is only responsible for pushing images of a certain project, it shouldn't naturally have:

- Global administrator capabilities
- Management capabilities for other projects

### Understanding to Establish at This Stage

Harbor security is not only about "can log in / cannot log in",  
but:

- What can be seen after login
- What can be modified after login
- Where can images be pushed to

---

## Part Seven: How to Understand Minimum Permissions on Kubernetes

This section is also very important.

As long as a pipeline can deploy K8s, it means it has some cluster access capabilities.  
At this point, we must think about:

### 1) Which Environments Can It Operate

For example:

- Can only modify `test`
- Cannot touch `prod`

### 2) Which Objects Can It Operate

For example:

- Can only update a certain Deployment
- Cannot modify resources across the entire cluster

### 3) Does It Have Excessive kubeconfig

If you directly give a pipeline an administrator-level kubeconfig, it's highly risky from a security perspective.

### Understanding to Establish at This Stage

In the future, whenever you see:

- CI / Jenkins / Shell scripts using `kubectl`

You should immediately think:

> What cluster permissions does this executor currently have? Is it the minimum necessary permission?

---

## Part Eight: How to Understand Minimum Permissions on Runner / Agent

Previously, in posts 14 and 15, you've learned:

- Runner / Agent are executors
- In Jenkins on K8s scenarios, Agent is often a Pod

Now add a layer of security understanding.

### Executors Are Not the More Capable, the Better

An executor that simultaneously has:

- Harbor push permissions
- Management permissions for all namespaces in K8s
- Permissions to delete resources
- Deployment permissions for all environments

Although it's "convenient", the risk is also very high.

### A More Reasonable Approach

Split executor capabilities by task, for example:

- Build executor: more biased toward Docker / Harbor
- Deploy executor: more biased toward kubectl / Helm
- Test executor: more biased toward verification and scanning

### Understanding to Establish at This Stage

The core idea of executor security is also minimum permissions:

- Which type of task uses which type of executor
- Which executors only get the capabilities they need

---

## Part Nine: What Is Artifact Security, Why Images Also Belong to Security Objects

This section is often overlooked by beginners.

You've been building and pushing images so far,  
but images are not ordinary neutral files; they will eventually be run in clusters.

Therefore, images belong to "artifacts" and are also security objects.

### Why Images Should Be Considered in Security

Because images may contain:

- High-risk vulnerabilities in old base images
- Unnecessary debugging tools
- Unnecessary root permission running methods
- Plaintext configurations
- Secret files
- Incorrect scripts or backdoor programs

### Understanding to Establish at This Stage

The basic meaning of "artifact security" is:

**Things published to the environment should be trustworthy.**

CI/CD security is not only about protecting "processes", but also about protecting "process outputs".

---

## Part Ten: The Most Worthwhile Awareness to Establish at This Stage for Artifact Security

At this stage, you don't need to immediately implement an enterprise-level security scanning platform,  
but at least establish 4 awareness points.

### Awareness 1: Don't Use Unclearly-Sourced Base Images

Previously, you've used:

- `nginx:1.27`
- `busybox:1.35`
- `alpine:3.20`

These at least have relatively clear sources.

### Awareness 2: Don't Put Sensitive Information in Images

For example:

- Plaintext passwords
- Private keys
- kubeconfig
- Token files

### Awareness 3: Simpler Images Are Better

The more bloated an image is, often:

- The larger the attack surface
- More convenient for debugging, but also higher risk

### Awareness 4: When Entering Harbor Advanced Topics Later, Gradually Get to Know Vulnerability Scanning Capabilities

This is the natural extension of artifact security from "awareness" to "platform capabilities".

---

## Part Eleven: Hands-on - Checking Which Locations in Current Experiment Are Most Likely to Leak Sensitive Information

This section doesn't require you to add new systems; just do a self-check.

### Step 1: Check Current Experiment Directory

Enter:

    cd ~/08-ci-cd/01-manual-release
    ls -la

Check for the presence of:

- Harbor login credential files
- kubeconfig files
- Plaintext password scripts
- Secret configuration files

### Step 2: Check Helm Directory

Enter:

    cd ~/08-ci-cd/08-helm-lab/manual-web
    ls -la

Confirm:

- No hardcoded passwords in values files
- No accidental inclusion of kubeconfig, token, etc. in the directory

### Understanding to Establish at This Step

Many security issues are not advanced attacks, but rather the simplest "just putting it in casually".

The most valuable habit to cultivate at this stage is:

**Do not place sensitive files randomly in the experiment directory.**

---

## Part 12: Hands-on - Checking for Obvious Unsafe Habits in Image Configuration

This section combines your most familiar Dockerfile to perform the simplest check.

### Step 1: Review Dockerfile

Enter the application directory:

    cd ~/08-ci-cd/01-manual-release
    cat Dockerfile

### What to Focus on Currently

1. Is the plain-text password `COPY` included?
2. Are unnecessary local files brought into the image?
3. Does the current image only contain what's truly needed for the experiment?

### Understanding to Establish at This Step

Although your current experiment image is very small and simple,  
this step is to build an intuition that will be useful in the future:

> Every file in the image must have a valid reason during the build process.

---

## Part 13: Reverse-Engineering from Current Mainline, CI/CD Security Minimal Checklist

From now on, it's recommended to ask yourself the following questions each time you perform a release experiment.

### 1) Are Harbor Credentials Randomly Placed?

- Are they written into scripts?
- Are they stored in repositories?
- Are they left behind locally?

### 2) Are K8s Access Credentials Too Broad?

- Is the current execution environment using an overly large kubeconfig?
- Is it possible to modify prod when only test should be deployed?

### 3) Is the Executor's Permission Too Broad?

- Does this Runner / Agent have too many capabilities?

### 4) Is the Image Itself Trustworthy?

- Is the base image clearly defined?
- Are unnecessary files included?
- Is the tag clear and traceable?

### Understanding to Establish at This Step

These 4 points form the minimal, most practical CI/CD security checklist at this stage.

---

## Part 14: Intentionally Creating a "Unsafe Habit" Anti-Example for Understanding

This section doesn't require you to actually submit passwords, but helps you identify what constitutes poor practices.

### Anti-Example 1: Directly Writing Harbor Password into Script

Example:

    docker login harbor.example.com -u admin -p MyPassword123

Issues:

- Easily appears in command history
- Password spreads with script files
- Difficult to manage later

### Anti-Example 2: Directly Putting kubeconfig into Project Directory

Issues:

- Easily accidentally committed
- Easily copied and spread
- Difficult to control permission boundaries

### Anti-Example 3: A Runner / Agent with Full Environment Deployment Permissions

Issues:

- Even a test pipeline error could affect prod

### Understanding to Establish at This Step

Security is often not about "complex encryption algorithms",  
but rather:

**Whether daily habits are creating risks.**

---

## Part 15: Recommended Security Habits to Form at This Stage

This section recommends you remember directly for future use.

### Habit 1: Do Not Place Plain-Text Sensitive Information in Experiments and Notes

### Habit 2: Prefer Machine Accounts for Machine-to-Machine Access

For example, prioritize Robot account thinking for Harbor.

### Habit 3: Executors Should Only Have Capabilities Needed for Current Tasks

### Habit 4: Image Tags Should Be Clear and Traceable

### Habit 5: Image Content Should Be As Simple and Explanatory as Possible

### Habit 6: Whenever You Can Deploy K8s, Ask Yourself "Is the Permission Too Broad?"

---

## Part 16: This Section's Practice Exercise

### Practice 1: Perform a Security Self-Check on Current Experiment Directory

Requirements:

- Check `~/08-ci-cd/01-manual-release`
- Check `~/08-ci-cd/08-helm-lab/manual-web`

Answers:

- Are there any sensitive files that shouldn't be placed?
- Are there unsafe script writing practices?

### Practice 2: List the Most Sensitive Accounts / Credentials in Current Mainline

At least list:

- Harbor Credentials
- K8s Access Credentials
- Future GitLab / Jenkins Credentials

And explain who should use them.

### Practice 3: Answer the Following 5 Questions

1. What does CI/CD Security Mainly Protect?
2. Why Should Credentials Not Be Written into Repositories?
3. Why Is Minimal Privilege a Core Principle?
4. Why Are Images Also Security Objects?
5. Why Is "Being Able to Run" Not Equivalent to "Being Able to Run Securely"?

If you can explain these 5 questions yourself, you've mastered this section.

---

## Content to Be Able to Explain After This Section

After completing this section, it's recommended to be able to clearly explain the following:

The foundation of CI/CD security is not just protecting platform pages, but protecting credentials, permissions, and artifacts throughout the entire release pipeline.  
Harbor login credentials, K8s access credentials, and Runner / Agent execution capabilities all belong to sensitive boundaries.  
One of the most important security principles is minimal privilege, meaning each account, executor, and process should only have the capabilities needed to complete the current task.  
At the same time, images as the final artifacts deployed to clusters also need to be trustworthy, without unnecessary sensitive information or unclear content.  
Therefore, truly secure CI/CD is not just "being able to build, push, and deploy", but "these actions being executed within appropriate permission boundaries by trustworthy artifacts".

## Common Issues and Troubleshooting Directions

### Issue 1: Do I Need to Care About Security if I'm Just Learning?

Yes.  
Currently, you don't need to set up a complete security platform, but you must first establish correct habits.  
Because many unsafe habits are hard to change once formed.

### Issue 2: Is Having More Permissions More Convenient?

Convenient in the short term, but with great long-term risks.  
One of the most typical mistakes in CI/CD security is giving pipelines excessive permissions for convenience.

### Issue 3: Is Artifact Security Only Concerned When Scanning on the Platform?

No.  
Platform scanning is an advanced capability later, but establishing the awareness that "images are also security objects" is important at this stage.

---

## Key Points to Master in This Section

1. What Are the Sensitive Boundaries in the Current Mainline?
2. What Is the Minimal Principle for Credential Management?
3. Why Is Minimal Privilege Important?
4. Why Are Images Also Security Objects?
5. What Are the Most Valuable Security Habits to Cultivate at This Stage?

## One-Sentence Summary

The foundation of CI/CD security isn't about making the system complex from the start, but about placing credentials in the right place, controlling permissions to the minimum, and ensuring the final deployed image is trustworthy.