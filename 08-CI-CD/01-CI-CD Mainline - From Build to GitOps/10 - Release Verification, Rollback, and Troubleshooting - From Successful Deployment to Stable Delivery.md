# 10 - Release Verification, Rollback, and Troubleshooting: From Successful Deployment to Stable Delivery

## Document Description

This article is the 10th note in the 08-CI-CD learning pathway.

In the previous articles 01 to 09, we have established a minimum functional deployment pipeline:

- Content changes
- Building images
- Designing image tags
- Pushing to Harbor
- Deploying via kubectl or Helm to K8s
- Observing Deployment rolling updates

However, this is not enough.

Because being able to deploy something and being able to deploy it stably are two different things.

In actual work, the most crucial skill in deployment often isn't:

- Knowing how to write `docker build`
- Knowing how to use `kubectl set image`
- Knowing how to perform `helm upgrade`

But rather these abilities:

- Knowing what to check after deployment
- Understanding when deployment is truly successful
- Being able to quickly rollback in case of issues
- Identifying where the problem might be in the pipeline based on observed symptoms

Therefore, the focus of this article is:

1. How to verify a successful deployment
2. How to perform Deployment and Helm rollbacks
3. How to diagnose and troubleshoot common issues
4. How to improve from being able to deploy to being able to deliver stably

This article continues to use the current experimental environment:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private repository
- `test` namespace as the experimental environment

## Tags

#Kubernetes #CI-CD #rollout #rollback #Helm #Deployment #troubleshooting #Harbor #stable delivery #practical notes

## Learning Objectives

After completing this article, you should be able to:

1. Understand which layers need to be checked for deployment verification
2. Realize why "Pod Running" does not equal "deployment successful"
3. Perform Deployment-level rollbacks
4. Perform Helm Release-level rollbacks
5. Quickly identify the stage of an issue based on common symptoms
6. Explain the troubleshooting sequence for a complete deployment pipeline

## Main Experimental Approaches in This Article

This article will focus on two main areas:

### Main Area 1: Deployment Verification

Confirm step by step whether the deployment is truly successful, from K8s status to business results.

### Main Area 2: Rollback and Troubleshooting After a Failed Deployment

By intentionally creating issues or using existing version history, we will complete:

- Deployment rollback
- Helm rollback
- Diagnosis of common failure scenarios

---

## Part 1: Establishing a Fixed Verification Sequence for “Deployment Verification”

Starting from this article, it is recommended that you follow a fixed sequence for verification after each deployment, rather than checking whatever comes to mind.

The recommended order is as follows:

1. Deployment layer
2. ReplicaSet layer
3. Pod layer
4. Log layer
5. Service/Access layer
6. Business content layer

This order is very important and will be used repeatedly later on.

## Part 2: Verifying the First Layer —— Deployment Layer

### Step 1: Check the Deployment Status

Execute:

    kubectl -n test get deploy
    kubectl -n test describe deploy manual-web

### Key Points to Focus On

Check these fields:

- `READY`
- `UP-TO-DATE`
- `AVAILABLE`

Also, check if there are any exceptions in `Events`.

### Understanding to Establish at This Step

The Deployment layer focuses on:

**Whether the update has reached the desired number of replicas and version from the controller's perspective.**

If this layer is not normal, there is no need to rush to check the business pages yet.

---

## Part 3: Verifying the Second Layer —— ReplicaSet Layer

### Step 1: Check the ReplicaSet

Execute:

    kubectl -n test get rs

### Key Points to Focus On

Check:

- Whether a new ReplicaSet has been created
- How the number of replicas in the new and old ReplicaSets has changed
- Whether the old version has been reduced to 0
- Whether the new version has reached the expected number of replicas

### Understanding to Establish at This Step

The ReplicaSet layer focuses on:

**Whether the rolling update of the Deployment has completed the version transition.**

If the old ReplicaSet is still present and the new ReplicaSet has not fully taken over, it means the deployment is still in progress or has encountered an issue.

---

## Part 4: Verifying the Third Layer —— Pod Layer

### Step 1: Check the Pod Status

Execute:

    kubectl -n test get pods -o wide

### Step 2: Check for Abnormal Pods

If a Pod is not normal, execute:

    kubectl -n test describe pod PodName

### Key Points toFor example, to roll back to revision 1:

    helm rollback manual-web 1 -n test

### Step 3: View the history again

Execute:

    helm history manual-web -n test

### Step 4: Observe the Pod update process

Execute:

    kubectl -n test get rs -w
    kubectl -n test get pods -w
    kubectl -n test rollout status deployment/manual-web

### Step 5: Verify the page content

Execute:

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

After entering, execute:

    wget -qO- http://manual-web.test.svc.cluster.local

### Understanding required for this step

Helm rollback restores:

**A specific configuration state from the Release history.**

It does not just revert the Deployment's image but restores the entire template and parameter settings of that particular version of the Release.

Therefore, Helm rollback has a higher level of granularity than Deployment rollback.

---

## Section 11: Differences between Deployment rollback and Helm rollback

This point is easily confused and needs to be clarified separately.

### Deployment rollback

What is rolled back:

- The history of a single Deployment template

Suitable for:

- When you directly use `kubectl set image`
- Or when the Deployment itself has undergone version updates

### Helm rollback

What is rolled back:

- The entire historical state of a Release

Suitable for:

- When you manage applications using Helm install / upgrade
- When you want all resources associated with the Chart to revert to an older state

### Understanding required for this step

You can remember it this way:

- Deployment rollback: More focused on individual workloads
- Helm rollback: More focused on application packages / Releases

---

## Section 12: Intentionally create a failure where the image does not exist

It is recommended to perform this exercise because it represents one of the most common release failures.

### Step 1: Intentionally update to a non-existent tag

Execute:

    kubectl -n test set image deployment/manual-web manual-web=your-Harbor-domain/test/manual-web:not-exist-999

### Step 2: Observe the Pod status

Execute:

    kubectl -n test get pods

### Step 3: View detailed events

Execute:

    kubectl -n test describe pod Pod-name

### Step 4: Check the rollout status

Execute:

    kubectl -n test rollout status deployment/manual-web

### Expected outcomes

You are likely to see:

- `ErrImagePull`
- `ImagePullBackOff`
- The rollout process gets stuck

### Understanding required for this step

This demonstrates that:

**The release process does not end just because the commands were executed successfully; real issues often become apparent during the image pull phase.**

It is also important to establish this troubleshooting reflex:

> When you see `ImagePullBackOff`, first check whether the tag actually exists on the Harbor page.

---

## Section 13: Intentionally create a failure where the page appears unchanged

This section is also valuable because it helps you understand why verification steps are crucial.

### Scenario explanation

Suppose you have built new content, but the Deployment does not actually reference the new tag, or the Helm values have not been updated correctly.

### Checking methods

#### 1) First, check the actual image of the Deployment

Execute:

    kubectl -n test describe deploy manual-web | grep -A3 Image

#### 2) If using Helm, first check the values file

Execute:

    cd ~/08-ci-cd/08-helm-lab/manual-web
    cat values.yaml

#### 3) Then, check the page content

Execute:

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh
    wget -qO- http://manual-web.test.svc.cluster.local

### Understanding required for this step

The most important conclusion here is that:

- Even if the content has been changed,
- If the image build was successful,
- And even if there is a new tag in Harbor,

it is not enough. The release is only considered complete when both the Deployment/Helm actually use the new tag and the business results have indeed changed.

---

## Section 14: Establish a fixed troubleshooting sequence for yourself

From now on, it is recommended that you follow this sequence whenever you troubleshoot issues.

### Step 1: Content verification phase

Check:

- `index.html`
- The source code
- What exactly has been changed

### Step 2: Image build verification phase

Check:

- Whether `docker build` was successful
- Whether the content is correct when run locally

### Step 3: Image storage verification phase

Check:

- Whether the tag exists on the Harbor page

### Step 4: Cluster configuration/release verification phase

If using kubectl- Observation during rollout
- Rollback
- Fault localization

When you are able to perform and explain these steps independently, then this main learning path of CI/CD has truly become part of your own knowledge system.