# 23-Argo CD Core Objects: Application, Project, and Repository

## Documentation Notes

This article is the 23rd note in the 08-CI-CD learning path.

The previous article completed the minimal installation and minimal observation of Argo CD. Now you already know:

- Argo CD is a GitOps controller
- It runs in K8s
- It is not a replacement for CI tools, but fills the layer of "continuous alignment to target state"

In this article, we begin to enter the three core objects that will be repeatedly encountered in Argo CD:

- Application
- Project
- Repository

These three objects need to be clarified first, otherwise you will easily get confused when looking at Argo CD pages later.  
This article continues to follow the approach of "minimal empty talk and closely align with current main line", focusing on:

1. First clarify what each of these three objects solves
2. Then understand them by combining with your existing Helm/values directories
3. Finally conduct a minimal observation experiment to establish "object relationship sense"

This article continues to align with the current environment:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private registry
- `argocd` namespace has Argo CD installed
- Already has `manual-web` Helm Chart and multi-environment values files

## Tags

#Kubernetes #CI-CD #GitOps #ArgoCD #Application #Project #Repository #Helm #I'llTakeYourNotes.

## Learning Objectives

After completing this article, you should master:

1. Understand what Application, Project, and Repository are
2. Understand the relationship between the three in Argo CD
3. Understand why Argo CD is not just "watching a single YAML file"
4. Be able to use your existing Helm Chart directory to understand the input source of Application
5. Be able to explain "why an application needs both configuration source and boundary assignment"

## This Article's Experiment Main Line

This article is divided into 4 sections:

1. First reverse-engineer from your existing Helm directory what Argo CD needs to manage
2. Then understand Repository, Application, and Project separately
3. Observe the positions of these objects in the Argo CD interface
4. Summarize the roles of these three in the subsequent GitOps main line

---

## Part One: Reverse-Engineering from Your Existing Helm Directory What Argo CD Needs to Manage

Return to your familiar Helm Chart directory:

    cd ~/08-ci-cd/08-helm-lab/manual-web
    ls
    ls templates
    cat values-dev.yaml
    cat values-test.yaml

You now have these things:

- `Chart.yaml`
- `templates/`
- `values.yaml`
- `values-dev.yaml`
- `values-test.yaml`

As explained in articles 19 and 22 earlier, this directory will naturally become:

**The target state configuration source in Git**

That is, what Argo CD will look at later is typically not "a randomly modified cluster state",  
but the configuration content in these Git repositories.

### Current Understanding to Establish

Argo CD actually manages, not the local directory on your computer,  
but:

- A path in a Git repository
- This path carries Helm/YAML/Kustomize etc. target states

This is what the Repository object will eventually take over.

---

## Part Two: What Is Repository and How to Understand It

In Argo CD, Repository can be initially understood as:

**The source repository for application target state configuration.**

Its most typical source is:

- Git repository

Sometimes it may also include:

- Helm repository

But focus on Git repositories first in this stage.

### Why Repository Is an Independent Object

Because Argo CD needs to first know:

- Where to get the configuration
- Whether this repository is trustworthy
- How to access this repository
- Whether this repository contains the application path you need

### Understanding in Combination with Your Current Main Line

In the future, your `manual-web` Chart is likely to be placed in a Git repository, for example:

- A repository specifically for K8s configurations
- Or a deployment directory in an application repository

Argo CD must first know this repository address before proceeding further.

### Current Understanding to Establish

Repository solves:

**Where does the configuration come from.**

---

## Part Three: What Is Application and How to Understand It

In Argo CD, Application can be initially understood as:

**A declaration of an application that needs to be continuously aligned to the cluster.**

It usually answers the following questions:

1. From which repository to get the configuration
2. Which path in the repository is the needed configuration
3. To which cluster to send it
4. To which namespace to send it
5. Whether this application is currently synchronized and healthy

### Why Application Is So Important

Because Argo CD is not "deploying the entire repository to the cluster in one go".  
It needs a clear object to represent:

- Which set of configurations
- Which actual application it corresponds to
- Where it ultimately lands

### Understanding in Combination with Your Current Main Line

In the future for `manual-web`, you may have:

- A dev environment Application
- A test environment Application

Even if they come from the same repository,  
they may:

- Use different paths
- Or the same path with different values
- Or target different namespaces

### Current Understanding to Establish

Application solves:

**Which application, with what target state, and where to deploy it.**

---

## Part Four: What Is Project and How to Understand It

In Argo CD, Project is not the concept of a "code project" in the ordinary sense.  
In this stage, you can initially understand it as:

**A set of boundaries and rules for a group of Applications.**

It more closely answers these questions:

- Which repositories are allowed within this scope
- Which target clusters/namespaces are allowed for deployment
- Which resource types are allowed to manage
- Which Applications belong to the same logical boundary

### Why Project Is Needed

Because without Project, Applications in Argo CD would easily become:

- Unbounded
- Unconstrained
- Able to accept any repository
- Able to deploy to any environment

This would become very chaotic in environments with multiple teams.

### Most Appropriate Understanding at This Stage

Initially understand Project as:

**The grouping and boundary controller for Applications.**

### Current Understanding to Establish

Project solves:

**What rules should these applications be subject to.**

---

## Part Five: Now Connect These Three Objects Together

This section is the most important main line of the entire article.

### Repository Resolution

Where does the configuration come from.

### Application Resolution

Which specific application, which configuration to send where.

### Project Resolution

Which sources can be used by this class of applications, which targets they can be sent to, and which boundaries they must follow.

### You can first remember it as a sentence

> Repository is the configuration source, Application is the target, and Project is the boundary rule.

### Current understanding to establish

When you look at Argo CD pages or YAML configurations later, just remember these three layers and you won't get confused:

1. First, there must be a configuration source
2. Then, there must be specific application declarations
3. Finally, there must be application boundary rules

---

## Part Six: Use the current `manual-web` main line to create a minimal example

Assume you have a Git repository in the future that contains:

- `manual-web` Helm Chart
- `values-dev.yaml`
- `values-test.yaml`

You can understand it this way.

### Repository

Is:

- This Git repository itself

### Application

Can be:

- `manual-web-dev`
- `manual-web-test`

These two Applications come from the same repository but have different targets.

### Project

Can be:

- A `learning` project
- Or a `platform-lab` project

It constrains:

- Only certain repositories can be used
- Only certain namespaces can be deployed to
- Only certain resources can be managed

### Current understanding to establish

The same Repository can be used by multiple Applications.  
Multiple Applications can also be grouped under the same Project.

---

## Part Seven: Why multiple Applications can exist in the same repository

This point is particularly important because it will frequently occur in later multi-environment scenarios.

For example, in the same Git repository, you may have:

- `apps/manual-web/`
- `apps/another-service/`

Or, under the same `manual-web` Chart, through different values to achieve:

- dev
- test
- prod

At this time:

- Repository is still the same
- But Applications can be multiple

### Current understanding to establish

Repository is not equal to Application.  
A repository may have:

- Multiple paths
- Multiple applications
- Multiple environments

Application is the definition of one "synchronizable target unit."

---

## Part Eight: Why Project is not optional

Although only one person is doing the experiment at this stage, you should still establish this feeling first.

Without Project, Application easily becomes:

- Anyone can connect to any repository
- Anyone can deploy to any namespace
- Boundaries become increasingly blurred

But once there is a Project, it's easier to enforce rules, for example:

- This Project only allows certain Git repositories
- This Project only allows deployment to `dev` and `test`
- Not allowed to touch `prod`
- Only certain resource types can be managed

### Current understanding to establish

The value of Project is not "adding another layer of hassle,"  
but rather:

**Giving Application boundaries.**

---

## Part Nine: Observing Repository entry in Argo CD page

This part begins with a minimal observation experiment.

### Step One: Confirm Argo CD page is still accessible

If your previous port-forward hasn't been started, re-execute:

    kubectl -n argocd port-forward svc/argocd-server 8080:443

Open browser to:

    https://127.0.0.1:8080

### Step Two: Find Repository-related entry in the page

It's usually located in:

- Settings
- Repositories

Or similar locations.

### Observation Points

Check if the page emphasizes:

- Connecting to repository
- Repository address
- Credentials
- Repository status

### Current understanding to establish

Repository exists separately in UI because Argo CD must first know "where the configuration source is."

---

## Part Ten: Observing Application entry in Argo CD page

### Step One: Find Applications page

This is usually one of the most prominent core pages in Argo CD UI.

### Observation Points

The interface usually revolves around these contents:

- Application name
- Sync status
- Health status
- Target cluster
- Target namespace

### Current understanding to establish

Application is the most core daily operation object in Argo CD.  
Because what's truly "aligned" is not the abstract repository, but individual Applications.

---

## Part Eleven: Observing Project entry in Argo CD page

### Step One: Find Project-related entry

It's usually located in:

- Settings
- Projects

Or similar locations.

### Observation Points

Check what Project page usually cares about:

- Allowed source repositories
- Allowed destinations
- Allowed resource scope

### Current understanding to establish

Project has less presence in the UI compared to Application,  
because it's more about "rule and boundary management,"  
rather than "daily sync result display."

---

## Part Twelve: Why not rush to create a remote Git Repository now

This point needs to be clearly explained to avoid disrupting your rhythm by pursuing completeness now.

### Reasons for not rushing to do this now

1. You need to first clarify the object relationships
2. Your local directory is already sufficient to help you understand the source of target state
3. Once you start connecting to real remote Git, it will also introduce:
   - Repository access
   - Credentials
   - Application configuration
   - Sync method
4. These are better suited for later articles to continue connecting

### Current understanding to establish

First establish the "object relationship mind map,"  
it's more important than immediately connecting everything.

---

## Part Thirteen: Map your current Helm directory to these three objects

This part is recommended to go through yourself.

Assume you put your current directory:

    ~/08-ci-cd/08-helm-lab/manual-web

into a Git repository.

Then:

### Repository

Is this Git repository.

### Application

Can be:

- `manual-web-dev`
- `manual-web-test`

### Project

Can be:

- `learning`
- Or a custom practice project boundary you define

### Current understanding to establish

By now, you should already be able to naturally say:

**Application is not the repository itself, but the definition of "which target state to take from the repository and where to send it."**

---

## Part 14: Recommended Minimum Understanding Scope for This Stage

Recommend that you fix the three concepts as follows:

### Repository

Where the configuration source comes from.

### Application

Which specific configuration to synchronize to which environment.

### Project

The repository source, target environment, and resource boundary rules for this set of Applications.

### Understanding to Establish at This Stage

This scope is sufficient to proceed to the following:

- Sync
- Diff
- Health
- Self-Heal
- App of Apps
- Multi-Environment GitOps

---

## Part 15: This Section's Practice Exercise

### Exercise 1: Create a Three-Object Example Using the Current `manual-web` Main Line

Requirements:

- Identify which is the Repository
- Which is the Application
- Which is the Project

### Exercise 2: Find the Three Types of Object Entries on the Argo CD Page

Requirements:

- Repository Entry
- Application Page
- Project Entry

And record the core fields you see.

### Exercise 3: Answer the Following 5 Questions Yourself

1. What problems do Repository, Application, and Project respectively solve?
2. Why can a single Repository correspond to multiple Applications?
3. Why is Application the actual object being synchronized?
4. Why is Project not optional?
5. Why must these three be understood separately?

If you can explain these 5 questions yourself, you've mastered this section.

---

## Content to Be Able to Explain After This Section

After completing this section, recommend that you can clearly explain the following:

In Argo CD, Repository, Application, and Project are three objects with different responsibilities.  
Repository solves "where the configuration comes from," typically corresponding to Git repositories or Helm repositories; Application solves "which configuration to synchronize to which environment," and it is the core synchronization object in Argo CD; Project solves "what boundary and rule constraints these Applications should follow," such as which repositories are allowed to use and which namespaces to deploy to.  
Only after separating these three can Argo CD manage configuration sources, application states, and boundary rules simultaneously.

## Common Questions and Troubleshooting Directions

### Question 1: Why Can't Repository and Application Be Treated as the Same Concept?

Because a single repository may contain multiple applications, multiple environments, and multiple paths.  
Application selects only one target state to synchronize from this repository.

### Question 2: Why Is Project Not as Intuitive as Application?

Because Project focuses more on boundary management, not the direct "application instance" operated daily.

### Question 3: Is This Section Less Important Since We Haven't Connected to a Real Git Repository Yet?

No.  
The focus of this section is to first clarify the object model, so that we won't get confused when connecting to repositories and synchronizing later.

---

## Key Takeaways for This Section

1. Basic definitions of Repository, Application, and Project
2. Relationships between the three
3. Why a single repository can have multiple applications
4. Why Project is the container for application boundary rules
5. How to map the current Helm directory to these three objects

## One-Sentence Summary

Argo CD can turn GitOps into a manageable system not by relying on a single object, but by using Repository to manage configuration sources, Application to manage synchronization targets, and Project to manage boundary rules.