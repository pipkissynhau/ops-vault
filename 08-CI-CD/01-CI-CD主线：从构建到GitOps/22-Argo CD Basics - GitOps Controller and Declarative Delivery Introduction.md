# 22-Argo CD Basics: GitOps Controller and Declarative Delivery Introduction

## Document Notes

This is the 22nd note in the 08-CI-CD learning path.

In the previous 19th note, we established the foundation of GitOps, with the core being:

- Git can serve as the source of desired state
- Cluster actual state may drift
- Need a controller to continuously align Git state and cluster state

In this note, we begin to implement this "controller" with a specific tool:

**Argo CD.**

This note doesn't require you to fully set up Argo CD in production today, nor does it expect you to master all object models from the start.  
The focus at this stage is:

1. Accurately position Argo CD within the entire pipeline
2. Understand its boundaries with previously learned tools like kubectl / Helm / GitLab CI / Jenkins
3. Perform a minimal installation and minimal observation experiment
4. First clarify what "GitOps controller is monitoring"

This document continues to align with the current main line and experimental environment:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private registry
- `test` namespace
- Completed minimal application `manual-web` build, push, kubectl / Helm deployment, rollback, and multi-environment experiments

## Tags

#Kubernetes #CI-CD #GitOps #ArgoCD #Helm #DeclaredDelivery #ConfigureAlignment #Controller #I'llTakeYourNotes.

## Learning Objectives

After completing this note, you should understand:

1. Argo CD's position in the entire delivery pipeline
2. The boundaries between Argo CD and GitLab CI / Jenkins / Helm
3. What are Argo CD's core responsibilities
4. Be able to complete a minimal installation in the current cluster
5. Be able to access the Argo CD interface and observe basic status
6. Be able to explain "why Argo CD is a GitOps controller, not just a replacement for kubectl as a deployment tool"

## This Note's Experimental Main Line

This note is divided into 4 sections:

1. Place Argo CD back into the entire pipeline you've already studied
2. Complete a minimal installation in the current cluster
3. Observe core resources after Argo CD installation
4. Use existing Helm / values directory to establish the feeling of "who will take over the target state in the future"

---

## Part 1: Place Argo CD Back into the Entire Pipeline

Previously, you've studied:

- GitLab CI / Jenkins: More focused on the front half automation
- Harbor: Store images
- Helm: Templating and parameterization configuration
- kubectl / Helm upgrade: Actively push configuration to the cluster
- GitOps: Git as the source of desired state

Now placing Argo CD into the pipeline, the most reasonable position is:

Code  
→ CI build image  
→ push Harbor  
→ update deployment configuration in Git  
→ Argo CD reads Git configuration  
→ Argo CD keeps cluster state continuously aligned with Git desired state

### Understanding to Establish at This Step

Argo CD is not a replacement for:

- Docker build
- Harbor push
- Image tag design

It mainly solves:

**Who will continuously align cluster state after images and configurations are ready.**

---

## Part 2: Understanding the Boundaries Between Argo CD and Previously Studied Tools

This section must be clearly explained, otherwise it's easy to get confused later.

### What GitLab CI / Jenkins Solves

More focused on:

- Pull code
- test
- build
- tag
- push Harbor
- Sometimes also trigger configuration changes

### What Helm Solves

More focused on:

- Templating
- Parameterization
- Chart organization
- values override

### What kubectl / helm upgrade Solves

More focused on:

- Actively execute a deployment command

### What Argo CD Solves

More focused on:

- Continuously monitor consistency between Git configuration and cluster state
- Synchronize when differences are found
- Keep desired state consistent long-term

### Understanding to Establish at This Step

Argo CD's focus is not "helping execute a command once",  
but:

**Monitor state and take responsibility for continuous alignment.**

---

## Part 3: Why It's Called "GitOps Controller"

You've already studied the general idea of Kubernetes controllers:

- Deployment controller monitors replica count and template state
- Adjusts continuously when state doesn't match target

Argo CD essentially has a similar "controller mindset", but it focuses more on:

- Desired state in Git
- Current state in the cluster

It will take action when inconsistencies are found.

### Understanding to Establish at This Step

Argo CD is called a GitOps controller instead of just a CLI tool  
because it's not "execute once", but:

- Continuously observe
- Continuously compare
- Continuously align

---

## Part 4: Confirm Current Cluster Status Before Installation

This section begins the hands-on work.

### Step 1: Confirm Nodes Are Normal

Execute:

    kubectl get nodes

### Step 2: Confirm Current Cluster Has Basic Capabilities

Execute:

    kubectl get ns
    kubectl get pods -A

### Step 3: Confirm Current Experimental Environment Has No Obvious Abnormalities

For example, check:

    kubectl -n test get deploy,pods,svc

### Understanding to Establish at This Step

Argo CD is to be installed into the current cluster,  
so before installation, confirm:

- The cluster itself is healthy
- Your own `test` environment has no unprocessed exceptions

---

## Part 5: Create a Dedicated Namespace for Argo CD

### Step 1: Create Namespace

Execute:

    kubectl create namespace argocd

If it prompts that it already exists, you can ignore it.

### Step 2: Confirm Namespace

Execute:

    kubectl get ns | grep argocd

### Understanding to Establish at This Step

Platform components like Argo CD are typically not recommended to be mixed with business applications in the same namespace.  
Using a separate `argocd` namespace is a natural approach.

---

## Part 6: Install Argo CD Official Manifests

### Step 1: Apply Official Installation Manifests

Execute:

    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

### Step 2: Observe Resource Creation Process

Execute:

    kubectl -n argocd get pods -w

### Expected Phenomenon

You'll see multiple pods starting up sequentially, for example common ones include:

- argocd-server
- argocd-repo-server
- argocd-application-controller
- argocd-dex-server
- argocd-redis

Names may vary slightly by version, but the overall pattern will be similar.

### Understanding to Establish at This Step

Argo CD is not a single Pod utility,  
it itself is also a set of platform components:

- Has a UI
- Has repository processing capabilities
- Has application control capabilities
- Has internal state and authentication-related components

---

## Part 7: Waiting for Argo CD Core Pod to be Ready

### Step 1: Check Pod Status

Execute:

    kubectl -n argocd get pods

### Objective

Wait until the main Pod enters:

- Running
- Ready

If some Pods remain abnormal, combine with describe to investigate.

### Step 2: Check Details if Necessary

Execute:

    kubectl -n argocd describe pod PodName

### Current Understanding to Establish

Installing Argo CD itself is a great exercise:

- It is also a set of K8s resources
- It will experience Pod creation, scheduling, image pulling, container startup
- The K8s troubleshooting approaches you learned earlier are still applicable here

---

## Part 8: View Core Resources Installed by Argo CD

This section is very important, not just for "knowing how to use", but for "understanding what it looks like in the cluster".

### Step 1: View Deployment

Execute:

    kubectl -n argocd get deploy

### Step 2: View Service

Execute:

    kubectl -n argocd get svc

### Step 3: View Other Resources

Execute:

    kubectl -n argocd get all

### Current Understanding to Establish

Argo CD is not a mysterious system outside the cluster,  
it itself is also an application running in K8s, albeit with the responsibility of:

- Managing the target state of other applications

---

## Part 9: Expose Argo CD Server, First Access the UI

The simplest way at this stage is to use port forwarding.

### Step 1: Port Forward argocd-server

Execute:

    kubectl -n argocd port-forward svc/argocd-server 8080:443

### Step 2: Browser Access

Open in your local browser:

    https://127.0.0.1:8080

Because it's local port forwarding + TLS, the browser may warn about certificate alerts. Proceed experimentally for now.

### Current Understanding to Establish

Now you can view Argo CD as a real control plane running in K8s, rather than just knowing the name.

---

## Part 10: Get Initial Admin Password and Login

### Step 1: Get Initial Password

Execute:

    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo

### Step 2: Login to UI

The username is usually:

    admin

The password is the value obtained from the previous step.

### Current Understanding to Establish

At this point, you have completed:

- Installing Argo CD
- Accessing UI
- Entering the control plane

Later, looking at GitOps won't feel abstract anymore.

---

## Part 11: Observe Argo CD UI, Don't Rush to Create Applications

At this stage, focus on the most important observations without rushing to use many features.

### What You Should Focus On

1. The location of the "application list" concept in the system
2. The location of the repository (repository) concept
3. Sync-related entries or status terms
4. Health status (health) related entries or status terms

### Current Understanding to Establish

You don't need to study all pages thoroughly right now.  
First know:

- Argo CD is organized around the "application state" and "sync state" concepts

This aligns with its role as a GitOps controller.

---

## Part 12: Treat Current Helm Directory as Future Target State Directory to be Managed by Argo CD

This section is very important because it connects the previous mainline with Argo CD's actual operation.

Enter your existing Helm directory:

    cd ~/08-ci-cd/08-helm-lab/manual-web

You now have:

- `Chart.yaml`
- `templates/`
- `values-dev.yaml`
- `values-test.yaml`

### Current Understanding of This Step

In the previous parts 8, 13, and 16, you have learned:

- Expressing application templates with Helm Charts
- Expressing environment differences with values
- Using `helm upgrade` to actively push state to the cluster

In a GitOps/Argo CD scenario, this directory can be further upgraded to:

**A target state source in Git.**

In other words, you don't need to manually run:

    helm upgrade ...

Instead:

- Modify values or Chart in Git
- Let Argo CD detect the changes
- Then have Argo CD synchronize

### Current Understanding to Establish

The Helm you learned earlier wasn't wasted,  
it's actually a common configuration input format for Argo CD.

---

## Part 13: Understanding "Declarative Delivery" at This Stage

This section's title includes "Declarative Delivery", which must be placed in a position you can understand.

### First Look at Your Previous Approach

For example:

    kubectl -n test set image deployment/manual-web manual-web=your Harbor domain/test/manual-web:v10

This is more like:

- I actively send a command to make the cluster change

### Now Look at GitOps/Argo CD Approach

This is more like:

- I declare in Git what this environment should look like
- The controller is responsible for making it that way and keeping it that way

### Current Understanding to Establish

Declarative delivery isn't just "fewer commands",  
it's:

**I care more about the final state rather than manually changing it step by step.**

---

## Part 14: Suggestions for Not Doing Anything at This Stage

This section is important to avoid getting into complexity too early.

### Things Not to Rush Into

- Don't rush to connect the entire Git repository
- Don't try to fully GitOps dev/test/prod today
- Don't rush to study all Argo CD objects
- Don't configure all sync strategies immediately

### Most Important Things at This Stage

1. Install it
2. Be able to see it
3. Know what it looks like in the cluster
4. Know what part of the mainline it will take over in the future

---

## Part 15: Now Look Back and Summarize, Why Is There Argo CD?

By this point, you should be able to answer this question.

### It's Not Because Previous Tools Are Inadequate

- GitLab CI/Jenkins still have value
- Harbor still has value
- Helm still has value
- kubectl still has value

### But Because There's Still a Missing Role

Missing is: /think

- Continuously monitor Git and cluster state differences  
- Detect drift  
- Perform synchronization  
- Ensure the target state remains consistent  

This is precisely where Argo CD fits.  

### Understanding to Establish at This Stage  

Argo CD's existence is not about reinventing the wheel,  
but rather filling in the missing layer along the entire workflow:  

**The layer of "continuous alignment with the target state."**  

---  

## Part 16: Practice Exercise in This Section  

### Exercise 1: Independently Complete an Argo CD Installation and Access  

Requirements:  

- Create `argocd` namespace  
- Apply the official installation manifests  
- Confirm main Pod readiness  
- Open the interface via port-forward  
- Retrieve the initial password and log in  

### Exercise 2: Observe Yourself What Core Components Argo CD Creates in the Cluster  

Requirements:  

- View `deploy`  
- View `svc`  
- View `pods`  
- Identify which components you consider most critical  

### Exercise 3: Answer the Following 5 Questions Yourself  

1. Where does Argo CD fit in the overall workflow?  
2. What is the boundary between Argo CD and GitLab CI / Jenkins?  
3. Why is Argo CD considered a controller rather than just a command tool?  
4. Why is Argo CD easily compatible with Helm?  
5. Why is Argo CD still needed even if you already know kubectl / Helm?  

If you can explain these 5 questions yourself, you've mastered this section.  

---  

## Content to Be Able to Explain After Completing This Section  

After finishing this section, it's recommended to be able to clearly explain the following:  

Argo CD is not a replacement for all previous deployment tools, but rather fills the controller role of "continuous alignment with the target state" in GitOps.  
GitLab CI and Jenkins lean more toward the earlier half of automation, handling build and push actions; Helm focuses more on templating and parameterized configuration; while Argo CD leans toward continuously reading the target state from Git and keeping the cluster state aligned with it.  
It itself is an application running in Kubernetes, installable in a dedicated namespace, observable via UI, and managed via controller patterns.  
Argo CD's core value lies not in adding another deployment command, but in making declarative delivery truly operational.  

## Common Issues and Troubleshooting Directions  

### Issue 1: Pods Don't Start After Installing Argo CD  

First, troubleshoot using familiar K8s methods:  

    kubectl -n argocd get pods  
    kubectl -n argocd describe pod PodName  

Focus on:  

- Image pull  
- Scheduling  
- Container startup failures  

### Issue 2: The Interface Won't Open  

First check:  

    kubectl -n argocd get svc  
    kubectl -n argocd port-forward svc/argocd-server 8080:443  

Confirm the port-forward process remains active.  

### Issue 3: Why Learn Argo CD If I Already Know Helm  

Because Helm addresses templating and parameters,  
while Argo CD further addresses:  

- Who continuously monitors the state  
- Who performs synchronization  
- Who handles drift  

---  

## Key Takeaways from This Section  

1. Argo CD's position in the main workflow  
2. The boundary between Argo CD and CI / Helm  
3. Why it's a GitOps controller  
4. How to complete a minimal installation in the current cluster  
5. How to treat existing Helm directories as future target state input sources  

## One-Sentence Summary  

Argo CD's core value is not adding another deployment entry point, but enabling the declarative target state in Git to be continuously aligned with Kubernetes clusters, turning GitOps from a concept into actual controller behavior.