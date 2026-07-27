# 19-GitOps Basics: Why Argo CD Still Exists

## Document Description

This article is the 19th note in the 08-CI-CD learning pathway.

Previously, in sections 01-18, you have successfully implemented a minimal deployment pipeline and gradually established the following components:

- Manual deployment
- Image building and tagging
- Harbor repository
- GitLab CI/Jenkins Pipeline
- K8s rolling updates
- Helm
- Multi-environment deployment
- Executors
- CI/CD security measures
- Advanced Harbor management

In this article, we will delve into a very important extended topic:

**GitOps.**

This section does not require you to immediately start using Argo CD or build the entire GitOps platform today.  
The focus is first on clarifying the following question:

> If we already have deployment methods such as GitLab CI, Jenkins, kubectl, and Helm, why do we still need Argo CD and GitOps?

If you don’t understand this clearly, you might think of Argo CD as “reinventing the wheel” when you encounter it later.  
But that’s not the case.

This article continues to align with your current learning path and experimental environment:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private repository
- `test` namespace
- You have already completed experiments on building, pushing, deploying, rolling back, and using Helm for the minimal application `manual-web`.

## Tags

#Kubernetes #CI-CD #GitOps #ArgoCD #Helm #kubectl #Declarative Deployment #Configuration Management #Practical Notes

## Learning Objectives

After completing this article, you should be able to:

1. Understand what GitOps aims to solve.
2. Recognize the differences between GitOps and traditional CI/CD approaches.
3. Comprehend why, after achieving “deployment capability,” there is still a need for “continuous alignment with target states.”
4. Grasp the role of Argo CD in the entire deployment pipeline.
5. Compare and understand the value of GitOps using the deployment methods you have already learned.
6. Clearly explain why Argo CD is necessary.

## Experimental Approach for This Article

In this section, we will not install Argo CD first. Instead, we will focus on three key tasks:

1. Identify the boundaries between the kubectl/Helm-based deployment methods you are familiar with.
2. Understand the core concept of GitOps: using Git as the source of target states.
3. Simulate the “Git-based target state” approach using your current experimental environment.

---

## Part 1: Review the Deployment Methods You Already Know

You have already explored two common deployment methods:

### Method 1: Direct kubectl Deployment

You have executed commands like:

    kubectl -n test set image deployment/manual-web manual-web=your-Harbor-domain/test/manual-web:v10

Or:

    kubectl apply -f manual-web.yaml

### Method 2: Deployment via Helm

You have used commands such as:

    helm upgrade manual-web . -n test

Or with custom values:

    helm upgrade manual-web-test . -f values-test.yaml -n test

### Key Understanding for This Step

You now have the ability to deploy applications.  
This is crucial because GitOps does not replace basic deployment capabilities; instead, it builds upon them to address additional challenges.

---

## Part 2: Identify Common Elements of These Deployment Methods

Whether you use kubectl or Helm, these methods share one common characteristic:

**The deployment action is initiated manually by you.**

In other words:

- You execute a command.
- K8s processes the command.
- The cluster state changes accordingly.

This approach can be described as **push-based deployment**, where changes are actively pushed into the cluster by humans or pipelines.

### Key Understanding for This Step

The previous deployment methods are valid and widely used.  
However, their core limitation is that:

- Changes are always initiated externally.
- The cluster does not automatically adjust its state to match a “declared target.”

This is where GitOps comes in handy.

---

## Part 3: Common Challenges with Current Deployment Methods

This section is crucial because GitOps was developed precisely to address these issues.

### Challenge 1: Inconsistent Cluster States

For example:

- You assume the `manual-web` Deployment should be at version `v10`.
- But someone manually changed it to `v9`.
- Or temporary fixes were applied directly in the cluster.
- As a result, Git, local files, and the actual cluster state become inconsistent.

### Challenge 2: Difficulty Tracing Changes

If many operations are performed through direct commands like:

- `kubectl edit`
- `kubectl set image`
- Manual `helm upgrade`

it can be difficult to determine:

- Which Git change caused the current state.
- Who madeYou already have half of the core components of GitOps:

- The target state configuration file

What's missing is the controller that continuously monitors and synchronizes these states.

---

## Part 9: Conducting a Manual Experiment to Simulate GitOps Alignment

This part is crucial.

We don't need Argo CD for now; we can manually simulate GitOps concepts.

### Step 1: Verify the Current Version in the Test Environment

Execute:

    kubectl -n test describe deploy manual-web | grep -A3 Image

Note down the current actual image tag.

### Step 2: Check the Target State in the Helm Values for the Test Environment

Execute:

    cd ~/08-ci-cd/08-helm-lab/manual-web
    cat values-test.yaml

Look at the following fields:

- `image.tag`
- `replicaCount`
- `namespace`

### Step 3: Intentionally Make the Cluster Inconsistent with values-test.yaml

For example, if `values-test.yaml` currently specifies:

    tag: "test-13"

You can manually change the Deployment image tag to something else, like this:

    kubectl -n test set image deployment/manual-web manual-web=your-Harbor-domain/test/manual-web:v18

### Step 4: Observe the Difference Again

Now:

- Git/Chart target state: Still `test-13`
- Cluster actual state: Has changed to `v18`

This is a simple example of "state drift."

### Step 5: Manually Re-execute Helm Upgrade to Restore the Original State

Execute:

    helm upgrade manual-web-test . -f values-test.yaml -n test

Then check again:

    kubectl -n test describe deploy manual-web | grep -A3 Image

### Key Points to Understand at This Stage

What you just did manually is essentially:

- Identifying when the cluster deviates from the target state
- Manually bringing it back in line with the target state

GitOps/Argo CD aim to automate and sustain this process.

---

## Part 10: What is "State Drift"

Now that we've covered this, explaining state drift becomes much clearer.

State drift means:

**The actual current state of the cluster deviates from the target state defined in Git.**

### Common Sources

- Manual changes to Deployments
- Emergency fixes applied directly in the cluster
- Rollouts not synchronized back to Git
- Temporary modifications to resource parameters in the environment

### Why State Drift is Dangerous

Once state drift occurs:

- Git can no longer reflect the true environment
- The environment becomes increasingly difficult to manage
- During troubleshooting, there may be different perceptions of the "current state"

### Key Points to Understand at This Stage

One of the benefits of GitOps is that it makes it easier to detect and even automatically correct state drift.

---

## Part 11: Why GitOps Is Particularly Suitable for Multiple Environments

As you learned in Articles 13 and 16:

- The differences between dev, test, and prod environments
- How values-dev.yaml and values-test.yaml are used for configuration management

GitOps fits perfectly here because:

### Each Environment Can Have Its Own Target State File

For example:

- `values-dev.yaml`
- `values-test.yaml`
- `values-prod.yaml`

### Each Environment Can Have Its Own Ideal Configuration State

GitOps controllers only need to know which configuration file should be used for each environment:

- dev
- test
- prod

This makes managing multiple environments much clearer.

### Key Points to Understand at This Stage

GitOps works well with Helm values because Helm values are naturally designed to represent the target state of different environments.

---

## Part 12: Understanding the Role of Argo CD at This Phase

Looking back at Argo CD now, it makes more sense.

You can think of it as:

**A controller that continuously monitors whether the Git state and the Kubernetes state are consistent.**

It typically performs the following tasks:

1. Checks the configuration in the Git repository
2. Inspects the actual resources in the cluster
3. Compares the two
4. Decides whether to perform synchronization
5. Adjusts the cluster state to match the target state

### Key Points to Understand at This Stage

Argo CD isn't responsible for building images or writing Dockerfiles. Instead, it acts as a continuous aligner of cluster configuration states.

---

## Part 13: How to Remember GitOps and Argo CD at This Phase

It's recommended to focus on the following basic definitions:

### GitOps

Is a philosophy for release and configuration management:

- Git serves as the source of the target state
- The cluster should always be aligned with the Git state

### Argo CD

Is a common implementation tool for this philosophy:

- It monitors Git and the cluster
- It performs synchronization when needed
- It helps detect and address state drift

### Key