# 27-Argo CD Troubleshooting Basics: Application Asynchrony, Resource Drift, and Sync Failures

## Document Description

This article is the 27th note in the 08-CI-CD learning pathway.

Previous articles have gradually covered the main aspects of Argo CD:

- Article 19: Why GitOps Still Matters
- Article 22: Minimum Installation and Access for Argo CD
- Article 23: The Three Core Objects: Application, Project, and Repository
- Article 25: How Argo CD Integrates with Helm
- Article 26: Using a Git Repository to Drive a Minimum Release

In this article, the focus shifts to:

**How to troubleshoot issues that arise with Argo CD.**

This article does not aim to provide a comprehensive guide to troubleshooting across all platforms but instead focuses on the most common issues based on the minimum setup you have already implemented.

1. Application Asynchrony
2. Resource Drift
3. Sync Failures
4. Successful Sync but Incorrect Business Results

The explanations will be tailored to operations in your current environment:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private repository
- `argocd` namespace with Argo CD installed
- Existing `manual-web` Helm Chart
- Minimum GitOps repository and test environment Application

## Tags

#Kubernetes #CI-CD #GitOps #ArgoCD #Troubleshooting #OutOfSync #SyncFailed #Helm #ResourceDrift #PracticalNotes

## Learning Objectives

After completing this article, you should be able to:

1. Understand why Argo CD troubleshooting needs to be structured in layers.
2. Define what OutOfSync, resource drift, and SyncFailed mean.
3. Reproduce a common resource drift issue in your current environment.
4. Identify where a sync failure might occur in your current setup.
5. Explain the recommended order of checking when troubleshooting Argo CD issues.

## Main Troubleshooting Structure for Argo CD

At this stage, Argo CD troubleshooting can be divided into 4 main layers:

1. Git / Repository Layer
2. Application Definition Layer
3. Rendering / Sync Layer
4. Cluster Operation and Business Results Layer

Once these layers are clearly defined, many troubleshooting issues will become easier to address.

---

## Part 1: Establishing a Fixed Order for Argo CD Troubleshooting

In Article 21, you learned about the general CI/CD troubleshooting process:

- build
- push
- deploy
- verify

Now that you are using Argo CD, the troubleshooting order needs to be adjusted to fit the GitOps framework.

### Recommended Current Order

### Layer 1: Git / Repository Layer

First, confirm:

- Whether the Git repository is accessible.
- Whether the target state in the repository has actually changed.
- Whether the push was successful.
- Whether Argo CD is viewing the correct repository content.

### Layer 2: Application Definition Layer

Next, verify:

- Whether the Repository URL is correct.
- Whether the Path is correct.
- Whether the values file specifications are accurate.
- Whether the destination namespace is correct.

### Layer 3: Rendering / Sync Layer

Then, check:

- Whether the Helm Chart can be rendered correctly.
- Why the synchronization failed.
- What specific step caused the sync failure.

### Layer 4: Cluster Operation and Business Results Layer

Finally, confirm:

- Whether the Deployment/Pod is functioning normally.
- Whether the rollout was successful.
- Whether the page content is correct.

### Understanding to Establish at This Step

Whenever an issue occurs with Argo CD, don’t immediately focus on the red status displayed in the UI. First, ask yourself:

> Is it a problem with Git, Application configuration, synchronization, or cluster operation?

---

## Part 2: Understanding the Three Most Common Status Terms

It is essential to understand these terms before proceeding with troubleshooting.

### 1) OutOfSync

This indicates that:

**The target state in Git does not match the current state of the cluster.**

It doesn’t necessarily mean a failure. It could mean one of the following:

- The Git state has changed, but the cluster hasn’t updated yet.
- The cluster state was manually modified, causing it to diverge from the Git state.

### 2) Synced

This means that:

**From Argo CD’s perspective, the current resource state in the cluster is aligned with the Git target state.**

Note that:

- Synced does not guarantee that there are no business issues.
- It merely indicates that the states have been matched successfully.

### 3) SyncFailed

This indicates that:

**Argo CD attempted to synchronize but failed.**

This usually means one of the following:

- There is an issue with the Application definition.
- There is a problem with Helm chart rendering.
### Step Two: Intentionally change the Helm values file to a non-existent name

For example, if it is originally:

- `values-test.yaml`

Change it to:

- `values-test-not-exist.yaml`

If you are currently managing the Application using YAML, you can also directly modify the Application resource definition.

### Step Three: Refresh or re-trigger synchronization

### Expected Outcome

You will most likely observe:

- The Application entering an error state
- Synchronization failure
- Related error messages indicating:
  - The values file could not be found
  - Helm rendering failed

### Understanding Required for This Step

This type of issue falls under:

**Application definition layer / Helm rendering layer problems**

It is neither a Harbor problem nor a Deployment-specific issue.

---

## Section Nine: Where to Start When You Encounter SyncFailed

At this stage, it is recommended that you follow this sequence:

### Step One: Check the synchronization error details in the Argo CD UI

Don’t just look at the “red” status;
click into it to see the specific error message.

Many times, you can directly find here:

- path not found
- values file not found
- manifest generation error
- permission denied

### Step Two: Return to the Git repository to check the target state source

Verify:

- Whether this file exists in the repository
- If the filename is correct
- If the directory path is accurate

### Step Three: If Helm input was used, verify locally with `helm template`

Enter the local copy of the repository directory:

    cd ~/08-ci-cd/26-argocd-gitops-repo/manual-web

Execute:

    helm template manual-web . -f values-test.yaml -n test

Or, if you intentionally entered incorrect information, you will encounter the corresponding error.

### Understanding Required for This Step

Many synchronization failures in Argo CD can actually be reversed-verified using the local Helm tools you are already familiar with.

---

## Section Ten: Practical Exercise – Creating a Minimum Synchronization Failure: Incorrect Target Namespace

This type of issue is also very common.

### Step One: Change the Application’s target namespace to an incorrect or non-existent value

For example:

- `test-not-exist`

### Step Two: Re-sync

### Expected Outcome

Depending on your configuration and environment, you may observe:

- The resource cannot be successfully created
- Synchronization failure
- Or the resource is not placed in the expected location

### Understanding Required for This Step

This type of issue usually pertains to:

**Application destination definition layer / Cluster apply layer problems**

Therefore, when synchronization fails, don’t just focus on the Chart;
also check:

- Where it was deployed
- Whether the target namespace is appropriate

---

## Section Eleven: Synchronization Successful, but Business Results Are Still Incorrect – How to Troubleshoot

This is a situation where many people get misled.

You may see:

- Argo CD indicates Synced
- Even the Health status shows Healthy

But the business page still displays incorrect information.

### Possible Reasons

1. The target state in Git was originally incorrect
2. The image.tag in values points to an incorrect version
3. The content of that tag in Harbor is different from what you expected
4. The build process for the page content itself is incorrect
5. The Service / access path does not correspond to the correct resources

### Recommended Troubleshooting Order

#### Step One: Check `values-test.yaml` in Git

    cat ~/08-ci-cd/26-argocd-gitops-repo/manual-web/values-test.yaml

Confirm:

- What exactly is written for image.tag

#### Step Two: Check the actual image used by the Deployment

    kubectl -n test describe deploy manual-web | grep -A3 Image

#### Step Three: Verify if this tag actually exists on the Harbor page

#### Step Four: Access and verify the page content within the cluster

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh
    wget -qO- http://manual-web.test.svc.cluster.local

### Understanding Required for This Step

“Synced” means that:

- The cluster state is consistent with the Git target state

But it does not mean that:

- The target state in Git necessarily represents the correct business version you expect

---

## Section Twelve: How to Connect Argo CD Troubleshooting to the Main CI/CD Troubleshooting Flow

This is one of the most crucial understandings in this chapter.

In the previous chapter 21, you already knew how to troubleshoot in the following order:

- Build
- Push
- Deploy
- Verify

Now that you have Argo CD, you should add two more steps to this sequence:

### The recommended troubleshooting flow now becomes:

1. Check if the Git repository content is correct
2. Verify if the Application definition is accurate
3. Determine whether Ar