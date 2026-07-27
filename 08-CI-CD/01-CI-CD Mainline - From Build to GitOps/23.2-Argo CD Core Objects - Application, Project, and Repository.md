# 23-Argo CD Core Objects: Application, Project, and Repository

## Documentation Overview

This document is the 23rd note in the 08-CI-CD learning pathway.

In the previous article, you completed the minimal installation and observation of Argo CD. By now, you should know:

- Argo CD is a GitOps controller.
- It runs within Kubernetes.
- It does not replace CI tools but rather adds the layer of "continuous alignment with target states."

In this article, we will delve into three core objects that you will frequently encounter with Argo CD:

- Application
- Project
- Repository

If you don't first understand these three objects clearly, it can be quite confusing when you later navigate Argo CD's interface.  

This article continues the approach of "explaining fewer concepts in detail and staying close to the current main topic." The focus will be on:

1. Clarifying what each of these three objects solves.
2. Understanding them in context with your existing Helm/values directories.
3. Conducting a minimal observation experiment to establish a clear understanding of their relationships.

This document remains aligned with the current environment:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private repository
- The `argocd` namespace has Argo CD installed.
- You already have the `manual-web` Helm Chart and multi-environment values files.

## Tags

#Kubernetes #CI-CD #GitOps #ArgoCD #Application #Project #Repository #Helm #Practical Notes

## Learning Objectives

After completing this article, you should be able to:

1. Define what Application, Project, and Repository are.
2. Understand their relationship within Argo CD.
3. Recognize why Argo CD is more than just "monitoring a single YAML file."
4. Use your existing Helm Chart directory to understand the source of Application configurations.
5. Explain why an application requires both configuration sources and defined boundaries.

## Main Experiment Outline for This Article

This article is divided into four sections:

1. Determine what Argo CD needs to manage based on your existing Helm directory.
2. Understand Repository, Application, and Project individually.
3. Locate these objects on the Argo CD interface.
4. Summarize their roles in the overall GitOps workflow.

---

## Section 1: Determining What Argo CD Needs Based on Your Existing Helm Directory

Let's return to the familiar Helm Chart directory you have:

    cd ~/08-ci-cd/08-helm-lab/manual-web
    ls
    ls templates
    cat values-dev.yaml
    cat values-test.yaml

You already have the following components:

- `Chart.yaml`
- `templates/`
- `values.yaml`
- `values-dev.yaml`
- `values-test.yaml`

As discussed in Articles 19 and 22, this set of directories will eventually become:

**The source of target state configurations in Git.**

In other words, what Argo CD will use for configuration is typically not the "random cluster state" but these contents stored in a Git repository.

### Understanding to Establish at This Step

What Argo CD actually manages is not the local directory on your computer but rather:

- A specific path within a Git repository.
- This path contains target states in formats like Helm, YAML, or Kustomize.

This is what the Repository object will handle later on.

---

## Section 2: Understanding What a Repository Is

In Argo CD, a Repository can be initially understood as:

**The source repository for application target state configurations.**

Its most common source is:

- A Git repository.

Sometimes it may also include:

- A Helm repository.

But for now, focusing on Git repositories is sufficient.

### Why a Repository Is an Independent Object

Argo CD needs to know:

- Where to retrieve the configuration.
- Whether this repository is trustworthy.
- How to access this repository.
- Whether this repository contains the application paths it needs.

### Understanding in Context with Your Current Setup

In the future, your `manual-web` Chart will likely be stored in a Git repository, such as:

- A repository dedicated to Kubernetes configurations.
- Or within a deployment directory of an application repository.

Argo CD must first know the address of this repository before it can proceed.

### Understanding to Establish at This Step

A Repository addresses the question of:

**Where the configuration comes from.**

---

## Section 3: Understanding What an Application Is

In Argo CD, an Application can be initially understood as:

**A declaration of a target application that needs to be continuously aligned with the cluster.**

It typically answers the following questions:

1. Which repository should I use for configuration?
2. Which path within the repository contains the required configuration?
3. To which cluster should I deploy this application?
4. To which namespace should I deploy it?
5. Is this application currently synchronized and### Step 1: Use `kubectl` to forward the port for `argocd-server`

Run the following command in your terminal:

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

Then open a browser and navigate to:

```http://127.0.0.1:8080
```

### Step 2: Locate the “Repository”-related section on the page

It is usually found in areas such as:

- Settings
- Repositories

or similar locations.

### Key Points to Note

Check if the page highlights information related to:

- Connecting to repositories
- Repository addresses
- Credentials
- Repository status

### Understanding Required for This Step

The reason why “Repository” is presented separately in the UI is that Argo CD needs to know first “where the configuration sources are located”.

---

## Section 10: Observing the “Application” section on the Argo CD page

### Step 1: Find the “Applications” page

This is usually one of the most prominent core pages in the Argo CD UI.

### Key Points to Note

The interface here typically focuses on the following aspects:

- Application name
- Sync status
- Health status
- Target cluster
- Target namespace

### Understanding Required for This Step

“Application” is the most central object for daily operations in Argo CD.  
This is because what is actually “aligned” is not abstract repositories but individual applications.

---

## Section 11: Observing the “Project” section on the Argo CD page

### Step 1: Locate the “Projects” related section

It is usually found in areas such as:

- Settings
- Projects

or similar locations.

### Key Points to Note

On the Project page, you will typically pay attention to:

- Allowed source repositories
- Allowed destinations
- Allowed resource scope

### Understanding Required for This Step

The reason why “Project” is not as prominent as “Application” on the page is that it mainly serves to manage “rules and boundaries,” rather than directly displaying the results of daily sync operations.

---

## Section 12: Reasons Why There’s No Urgency to Create a Real Remote Git Repository at This Stage

It is important to clarify this point to avoid disrupting your progress in pursuit of completeness.

### Reasons for Not Rushing at This Stage

1. Your primary focus should be on clarifying the relationships between these objects first.
2. The local directory you currently have is sufficient for understanding where the target states come from.
3. Once you start connecting to a real remote Git repository, additional complexities such as:
   - Repository access
   - Credentials
   - Application configuration
   - Sync methods
   - will arise.
4. It is better to address these issues in subsequent chapters.

### Understanding Required for This Step

Building a “mind map of object relationships” at this stage is more crucial than immediately establishing all connections.

---

## Section 13: Mapping Your Current Helm Directory to These Three Objects

It is highly recommended that you go through this process yourself.

Suppose you decide to store your current directory:

```bash
~/08-ci-cd/08-helm-lab/manual-web
```

in a Git repository. Then:

- The **Repository** would be this Git repository itself.
- The **Applications** could include:
  - `manual-web-dev`
  - `manual-web-test`
- The **Project** could be:
  - `learning`
  - or any custom project boundary you define.

### Understanding Required for This Step

By now, you should be able to clearly state that:

**An “Application” is not the repository itself, but rather defines which configuration state from the repository should be synchronized to a specific destination.**

---

## Section 14: Recommended Minimum Level of Understanding at This Stage

It is suggested that you establish the following understanding for these three concepts:

- **Repository**: Indicates where the configuration sources come from.
- **Application**: Specifies which specific configuration should be synchronized to which environment.
- **Project**: Defines the source repositories, target environments, and resource boundary rules for a group of applications.

### Understanding Required for This Step

This level of understanding is sufficient to help you proceed with subsequent topics such as:

- Syncing configurations
- Comparing differences
- Checking health status
- Self-healing mechanisms
- Managing multiple environments through GitOps

---

## Section 15: Practice Exercises for This Chapter

### Exercise 1: Provide an example of the three objects using the current `manual-web` branch

Requirements:

- Identify which one is the Repository.
- Identify which one is the Application.
- Identify which one is the Project.

### Exercise 2: Locate the entries for the three types of objects on the Argo CD page

Requirements:

- Find the Repository entry.
- Access the Applications page.
- Navigate to the Project entry.

Record