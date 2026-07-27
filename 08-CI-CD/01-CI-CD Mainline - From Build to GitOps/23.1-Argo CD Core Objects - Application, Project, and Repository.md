# 23-Argo CD Core Objects: Application, Project, and Repository

## Documentation Overview

This document is the 23rd note in the 08-CI-CD learning pathway.

In the previous article, you completed the minimal installation and basic observation of Argo CD. By now, you should know:

- Argo CD is a GitOps controller.
- It runs within Kubernetes.
- It does not replace CI tools but adds the layer of "continuous alignment with target states."

In this article, we will delve into three core objects that you will frequently encounter with Argo CD:

- Application
- Project
- Repository

If you don't understand these objects first, it can be quite confusing when you start using Argo CD's interface.  
This article continues to follow the approach of "explaining less but more directly related to the current topic," focusing on:

1. Clearly explaining what each of these three objects addresses.
2. Using your existing Helm/values directories to understand them better.
3. Conducting a minimal observation experiment to build a sense of how these objects interact.

This document still uses the following environment setup:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private repository
- The `argocd` namespace with Argo CD installed
- Existing `manual-web` Helm Chart and multi-environment values files

## Tags

#Kubernetes #CI-CD #GitOps #ArgoCD #Application #Project #Repository #Helm #Practical Notes

## Learning Objectives

After completing this article, you should be able to:

1. Understand what Application, Project, and Repository are.
2. Recognize their relationships within Argo CD.
3. Comprehend why Argo CD is more than just tracking a single YAML file.
4. Use your existing Helm Chart directory to understand where Application configurations come from.
5. Clearly explain why an application requires both configuration sources and defined boundaries.

## Main Experimental Approach for This Article

This article is divided into four sections:

1. Starting from your existing Helm directory, figure out what Argo CD needs to manage.
2. Understand Repository, Application, and Project individually.
3. Observe the locations of these objects on the Argo CD interface.
4. Summarize their roles in the overall GitOps workflow.

---

## Section 1: Identifying What Argo CD Requires Based on Your Existing Helm Directory

Let's revisit your familiar Helm Chart directory:

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

In other words, what Argo CD will use is not the current cluster state but these configuration files stored in a Git repository.

### Key Understanding for This Step

Argo CD actually manages not your local directory but:

- A specific path within a Git repository.
- This path contains target state configurations in formats like Helm, YAML, or Kustomize.

This is what the Repository object will handle later on.

---

## Section 2: What is a Repository and How to Understand It Initially

In Argo CD, a Repository can be initially understood as:

**The source repository for application target state configurations.**

Its most common source is:

- A Git repository.

Sometimes it may also include:

- Helm repositories.

But for now, focusing on Git repositories is sufficient.

### Why a Repository Is an Independent Object

Argo CD needs to know:

- Where to retrieve the configurations.
- Whether this repository is reliable.
- How to access this repository.
- Which application paths are available in this repository.

### How to Apply This to Your Current Setup

In the future, your `manual-web` Chart will likely be stored in a Git repository, such as:

- A dedicated repository for Kubernetes configurations.
- Or a deployment directory within an application repository.

Argo CD must first know the address of this repository before it can proceed.

### Key Understanding for This Step

A Repository addresses the question of:

**Where the configurations come from.**

---

## Section 3: What is an Application and How to Understand It Initially

In Argo CD, an Application can be initially understood as:

**A declaration of a target application that needs to be continuously aligned with the cluster state.**

It typically answers the following questions:

1. Which repository should I use to retrieve configurations?
2. Which path within the repository contains the required configurations?
3. To which cluster should I deploy these configurations?
4. To which namespace should I deploy them?
5. Is this application currently synchronized and in good health?

###🔤 Translate the following text from Chinese_simplified to English. Preserve the original formatting, sentence structure, and terminology. Ensure that every word and sentence is translated as closely as possible to the original meaning, without summarizing or omitting any part of the content. The translation must be faithful, detailed, and maintain the original length and complexity. Deliver the translation exactly as required, without any additional commentary or explanation, and ensure the 🔤 symbols are removed in the final output.

kubectl -n argocd port-forward svc/argocd-server 8080:443

Open a browser and navigate to:

https://127.0.0.1:8080

### Step 2: Locate the Repository-related section on the page

It is usually found in:

- Settings
- Repositories

or a similar location.

### Key Points to Observe

Check if the page emphasizes the following:

- Connecting to repositories
- Repository addresses
- Credentials
- Repository status

### Understanding Required for This Step

The reason why Repositories are presented separately in the UI is that Argo CD needs to know “where the configuration sources are” first.

---

## Section 10: Observing the Application section on the Argo CD page

### Step 1: Locate the Applications page

This is usually one of the most prominent core pages in the Argo CD UI.

### Key Points to Observe

The interface here typically covers the following:

- Application name
- Sync status
- Health status
- Target cluster
- Target namespace

### Understanding Required for This Step

Applications are the most central objects for daily operations in Argo CD.  
What is actually “aligned” is not abstract repositories but individual Applications.

---

## Section 11: Observing the Project section on the Argo CD page

### Step 1: Locate the Project-related section

It is usually found in:

- Settings
- Projects

or a similar location.

### Key Points to Observe

On the Project page, you will typically focus on:

- Allowed source repositories
- Allowed destinations
- Resource scope allowed

### Understanding Required for This Step

Projects are less prominent than Applications on the page because they are more about “rule and boundary management,” rather than “displaying daily synchronization results.”

---

## Section 12: Reasons Not to rush into creating a real remote Git Repository at this stage

It is important to clarify this point to avoid disrupting your progress in pursuit of completeness.

### Reasons Not to Proceed Immediately

1. You need to first sort out the object relationships.
2. The local directory is currently sufficient for understanding the sources of target states.
3. Once you start connecting to a real remote Git repository, it will introduce additional complexities such as:
   - Repository access
   - Credentials
   - Application configuration
   - Synchronization methods
4. These aspects are better addressed in subsequent sections.

### Understanding Required for This Step

Building an “object relationship mind map” now is more important than immediately establishing all connections.

---

## Section 13: Mapping your current Helm directory to these three objects

It is recommended that you go through this process yourself.

Suppose you eventually move the current directory:

    ~/08-ci-cd/08-helm-lab/manual-web

into a Git repository.

Then:

### Repository

will be this Git repository.

### Application

can be:

- `manual-web-dev`
- `manual-web-test`

### Project

can be:

- `learning`
- Or a custom boundary for your practice project

### Understanding Required for This Step

By now, you should be able to clearly state that:

**An Application is not the repository itself but defines “which target state to retrieve from the repository and where to send it.”**

---

## Section 14: Recommended minimum understanding at this stage

It is suggested that you establish these three concepts based on the following definitions:

### Repository

Where the configuration sources come from.

### Application

Which specific configuration will be synchronized to which environment.

### Project

The source repository, target environment, and resource boundary rules for this set of Applications.

### Understanding Required for This Step

This understanding is sufficient to proceed with the subsequent topics such as:

- Sync
- Diff
- Health
- Self-Heal
- App of Apps
- Multi-environment GitOps

---

## Section 15: Practice Exercises for This Chapter

### Exercise 1: Provide an example of the three objects using the current `manual-web` mainline

Requirements:

- Identify which one is the Repository.
- Identify which one is the Application.
- Identify which one is the Project.

### Exercise 2: Locate the entrances for the three types of objects on the Argo CD page

Requirements:

- Repository entrance
- Application page
- Project entrance

And record the key fields you see.

### Exercise 3: Answer the following 5 questions yourself

1. What problems do Repository, Application,