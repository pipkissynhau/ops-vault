# 10 - Release Verification, Rollback, and Common Issue Troubleshooting: From Successful Deployment to Stable Delivery

## Document Notes

This article is the 10th note in the 08-CI-CD learning path.

The previous notes 01 to 09 have already established a minimal runnable release pipeline:

- Content changes
- Build image
- Design image Tag
- Push to Harbor
- Deploy to K8s via kubectl or Helm
- Observe Deployment rolling update

But this is still not enough.

Because "being able to release" and "being able to stably release" are two different things.

In actual work, the most critical ability for release is often not:

- Knowing how to write `docker build`
- Knowing how to write `kubectl set image`
- Knowing how to write `helm upgrade`

But rather these abilities:

- Knowing what to check after release
- Knowing when a release is truly successful
- Being able to quickly rollback if issues occur
- Being able to judge where in the pipeline a problem might be

So the focus of this article is:

1. How to verify after release
2. How to perform Deployment rollback and Helm rollback
3. How to identify and troubleshoot common failure symptoms
4. How to elevate "being able to release" to "being able to stably deliver"

This article continues to align with the current experimental environment:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private registry
- `test` namespace as the experimental environment

## Tags

#Kubernetes #CI-CD #rollout #rollback #Helm #Deployment #FaultCheck. #Harbor #SteadyDelivery #I'llTakeYourNotes.

## Learning Objectives

After completing this article, you should master:

1. Understanding what layers to check for release verification
2. Understanding why "Pod Running" does not equal "release success"
3. Being able to perform Deployment-level rollback
4. Being able to perform Helm Release-level rollback
5. Being able to quickly identify the stage of a problem based on common symptoms
6. Being able to explain the fault-troubleshooting order for a complete release pipeline

## This Article's Experimental Mainline

This article focuses on two mainlines:

### Mainline 1: Release Verification

From K8s status to business results, verifying layer by layer whether this release was truly successful.

### Mainline 2: Rollback and Troubleshooting After Release Failure

By intentionally creating problems or based on existing version history, completing:

- Deployment rollback
- Helm rollback
- Common failure symptom troubleshooting

---

## Part 1: Establish a Fixed Verification Order for Release Validation

Starting from this article, it's recommended to follow a fixed verification order after each release, rather than checking things as you think of them.

Recommended order:

1. Deployment layer
2. ReplicaSet layer
3. Pod layer
4. Log layer
5. Service / Access layer
6. Business content layer

This order is very important and will be used repeatedly later.

## Part 2: Verifying the First Layer — Deployment Layer

### Step 1: Check Deployment Status

Execute:

    kubectl -n test get deploy
    kubectl -n test describe deploy manual-web

### What to Focus On

Check these fields:

- `READY`
- `UP-TO-DATE`
- `AVAILABLE`

Also check if `Events` has any anomalies.

### Understanding to Establish at This Step

The Deployment layer focuses on:

**Whether, from the controller's perspective, this update has reached the target replica count and target version.**

If this layer is abnormal, there's no need to hurry to check the business page.

---

## Part 3: Verifying the Second Layer — ReplicaSet Layer

### Step 1: Check ReplicaSet

Execute:

    kubectl -n test get rs

### What to Focus On

Check:

- Whether a new ReplicaSet has appeared
- How the replica counts of new and old ReplicaSets change
- Whether the old version has been scaled to 0
- Whether the new version has reached the expected replica count

### Understanding to Establish at This Step

The ReplicaSet layer focuses on:

**Whether the rolling update of the Deployment has completed the version handover.**

If you see the old ReplicaSet still running and the new ReplicaSet hasn't fully started, it indicates the release is still in progress or has stalled.

---

## Part 4: Verifying the Third Layer — Pod Layer

### Step 1: Check Pod Status

Execute:

    kubectl -n test get pods -o wide

### Step 2: Check Details of Abnormal Pods

If a Pod is abnormal, execute:

    kubectl -n test describe pod PodName

### What to Focus On

Check if the Pod is in:

- Running
- Pending
- ImagePullBackOff
- ErrImagePull
- CrashLoopBackOff

### Understanding to Establish at This Step

The Pod layer focuses on:

**Whether the new version image has actually been pulled down and whether the container has actually started.**

This is a very critical layer in release verification, as many issues get stuck here.

---

## Part 5: Verifying the Fourth Layer — Log Layer

### Step 1: Check Container Logs

Execute:

    kubectl -n test logs PodName

If the Pod has multiple containers, add the container name; the current experiment typically has only one container.

### What to Focus On

Check:

- Whether nginx or application processes have errors
- Whether configuration loading is abnormal
- Whether the application exits immediately after startup
- Whether there are obvious abnormal logs

### Understanding to Establish at This Step

The log layer focuses on:

**Whether the application internally is normal, even though the container is running.**

In many cases:

- The Pod is Running
- The rollout was successful
- But the logs already show obvious errors

So the log layer cannot be skipped.

---

## Part 6: Verifying the Fifth Layer — Service and Access Layer

### Step 1: Check Service

Execute:

    kubectl -n test get svc
    kubectl -n test describe svc manual-web

### Step 2: Enter the Cluster to Verify Access

Execute:

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

After entering, execute:

    wget -qO- http://manual-web.test.svc.cluster.local

### What to Focus On

Confirm:

- Whether the Service exists
- Whether the Service selector matches the Pod
- Whether internal cluster access can retrieve results

### Understanding to Establish at This Step

This layer focuses on:

**Whether the K8s resource layer has actually connected the "running Pod" to the access path.**

If the Pod is normal but the Service is unreachable, the release cannot be considered successful.

---

## Part 7: Verifying the Sixth Layer — Business Content Layer

### Step 1: Check Page Content or Interface Results /think

Still within `curl-test` container, focus on whether the returned content contains the expected new version, for example:

    version: v9

### Understanding at this step

This is the final layer.

Previously, Deployment, ReplicaSet, Pod, and Service all being normal does not guarantee the business result is correct.

Therefore, the final layer of release validation is:

**Is the business result the version you intended to release.**

This is why "Pod Running" cannot equal "release success."

---

## Part 8: Understanding "Release Success" with a Fixed Perspective

From now on, it's recommended to break down "release success" into two layers.

### Layer 1: K8s Controller Perspective

Execute:

    kubectl -n test rollout status deployment/manual-web

If you see:

    deployment "manual-web" successfully rolled out

Explanation:

- From the Deployment controller's perspective, this update has converged

### Layer 2: Business Result Perspective

Confirm via `wget` or curl that the page or API interface indeed returns the expected version.

### Understanding at this step

So in the future, do not just say:

"Rollout succeeded, so release succeeded."

A more rigorous expression should be:

- Rollout succeeded
- Pod normal
- Service normal
- Business result correct

Only when all four conditions are met does it more closely approach "true release success."

---

## Part 9: Rollback with Deployment Method

This section targets the Deployment you previously updated with `kubectl set image`.

### Step 1: View Deployment Rollout History

Execute:

    kubectl -n test rollout history deployment/manual-web

### Expected Phenomenon

You'll see revision information.

### Step 2: Execute Rollback

Execute:

    kubectl -n test rollout undo deployment/manual-web

### Step 3: Observe Rollback Process

Execute:

    kubectl -n test get rs -w
    kubectl -n test get pods -w
    kubectl -n test rollout status deployment/manual-web

### Step 4: Verify Page Content Returns to Previous Version

Execute:

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

After entering, execute:

    wget -qO- http://manual-web.test.svc.cluster.local

### Understanding at this step

The essence of Deployment rollback is not "magic recovery," but:

**Rolling back the Deployment template to the previous version and re-triggering a rolling update.**

In other words, rollback itself is an update action, but the target version becomes the old version.

---

## Part 10: Rollback with Helm Method

If you've completed Helm install/upgrade in Parts 8 and 9, this section is essential.

### Step 1: View Release History

Execute:

    helm history manual-web -n test

### Step 2: Rollback to an Old Revision

For example, rollback to revision 1:

    helm rollback manual-web 1 -n test

### Step 3: Recheck History

Execute:

    helm history manual-web -n test

### Step 4: Observe Pod Update Process

Execute:

    kubectl -n test get rs -w
    kubectl -n test get pods -w
    kubectl -n test rollout status deployment/manual-web

### Step 5: Verify Page Content

Execute:

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

After entering, execute:

    wget -qO- http://manual-web.test.svc.cluster.local

### Understanding at this step

Helm rollback reverts:

**A specific version of the Release's configuration state.**

It's not just rolling back the Deployment's image, but reverting the entire Release's template and parameter results from that version.

Thus, Helm rollback operates at a higher level than Deployment rollback.

---

## Part 11: Difference Between Deployment Rollback and Helm Rollback

This distinction is easy to confuse, so it's important to address it separately.

### Deployment Rollback

Rollback Target:

- Single Deployment template history

Suitable for:

- You directly used `kubectl set image`
- Or the Deployment itself underwent version updates

### Helm Rollback

Rollback Target:

- Entire Release's historical state

Suitable for:

- You manage the application via Helm install/upgrade
- You want all Chart-related resources to revert to the previous state

### Understanding at this step

You can remember it this way:

- Deployment rollback: More focused on single workload object level
- Helm rollback: More focused on application package/Release level

---

## Part 12: Intentionally Create a Fault with Non-Existent Image

This section is recommended to perform once, as it's almost the most common release failure.

### Step 1: Intentionally Update to a Non-Existent Tag

Execute:

    kubectl -n test set image deployment/manual-web manual-web=your Harbor domain/test/manual-web:not-exist-999

### Step 2: Observe Pod Status

Execute:

    kubectl -n test get pods

### Step 3: Check Detailed Events

Execute:

    kubectl -n test describe pod Pod name

### Step 4: Check Rollout Status

Execute:

    kubectl -n test rollout status deployment/manual-web

### Expected Phenomenon

You'll likely see:

- `ErrImagePull`
- `ImagePullBackOff`
- Rollout stuck

### Understanding at this step

This indicates:

**Release isn't over just because the command executed successfully. True issues often only surface during the Pod's image pull phase.**

You should also establish a critical troubleshooting reflex:

> See `ImagePullBackOff` first, then check if the tag exists on the Harbor page.

---

## Part 13: Intentionally Create a "Page Didn't Change" Fault

This section is also very valuable because it helps you understand "why verification layers are important."

### Scenario Description

Assume you built new content, but the Deployment actually didn't reference the new tag, or the Helm values weren't updated successfully.

### Verification Methods

#### 1) Check the Deployment's actual image

Run:

    kubectl -n test describe deploy manual-web | grep -A3 Image

#### 2) If using Helm, check the values

Run:

    cd ~/08-ci-cd/08-helm-lab/manual-web
    cat values.yaml

#### 3) Check the page content

Run:

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh
    wget -qO- http://manual-web.test.svc.cluster.local

### Key Understanding at This Step

The most important conclusion here is:

- Content was changed
- The image was built
- Even if Harbor has the new tag

Still isn't enough.

Only when the Deployment/Helm actually references the new tag, and the business results truly changed, does the release count as completed.

---

## Part 14: Establish a Fixed Troubleshooting Order for Yourself

From now on, it's recommended to follow this order for troubleshooting in the future.

### Step 1: Content Verification

Check:

- `index.html`
- Source code
- What exactly you changed

### Step 2: Image Build Verification

Check:

- Whether `docker build` succeeded
- Whether the content is correct after local run

### Step 3: Image Repository Verification

Check:

- Whether this tag exists on the Harbor page

### Step 4: Cluster Configuration/Deployment Verification

If using kubectl:

- Whether the Deployment's image was actually changed

If using Helm:

- Whether the values were actually changed
- Whether `helm template` rendering is correct
- Whether `helm history` records this change

### Step 5: Pod Runtime Verification

Check:

- `kubectl get pods`
- `kubectl describe pod`
- `kubectl logs`

### Step 6: Service and Access Verification

Check:

- Whether the Service selector is correct
- Whether internal cluster access is normal

### Step 7: Business Result Verification

Check:

- Whether the page content/API results match the expected new version

### Key Understanding at This Step

Once this order is established, many CI/CD issues won't seem chaotic anymore.

Because you know:

**The problem isn't "the pipeline failed," but rather a segment of the chain failed.**

---

## Part 15: This Section's Practice Exercises

### Exercise 1: Perform a Full Deployment Rollback

Requirements:

- First publish a new version
- Then execute `rollout undo`
- Then verify the page rollback

### Exercise 2: Perform a Full Helm Rollback

Requirements:

- Use Helm upgrade to change the version once
- Check `helm history`
- Then rollback to the previous revision
- Verify the page rollback

### Exercise 3: Intentionally Create a "Image Not Found" Fault and Locate It

Requirements:

- Change the image to a non-existent tag
- Observe Pod status
- Use `describe pod` to check Events
- Explain which segment of the chain the issue belongs to

### Exercise 4: Answer the Following 6 Questions Yourself

1. What layers must be checked for release verification at least?
2. Why is Pod Running not equivalent to a successful release?
3. What does Deployment rollback revert?
4. What does Helm rollback revert?
5. Where should you check first for `ImagePullBackOff`?
6. Why should release failures be investigated by segmenting the chain?

If you can explain these 6 questions yourself, you've mastered this section.

---

## Content You Should Be Able to Explain After This Section

After completing this section, it's recommended to be able to explain the following:

After a release is completed, you shouldn't only check if the Deployment was updated or if the Pod is Running, but instead verify the Deployment, ReplicaSet, Pod, logs, Service, and final business results in a fixed order.  
True release success includes both the K8s controller's rollout success and the business results actually being the target version.  
If the new version has issues, you can roll back using Deployment rollback or Helm rollback, where Deployment rollback reverts the history of a single workload template, and Helm rollback reverts the entire Release's historical state.  
When a fault occurs, you shouldn't generically say the pipeline failed, but instead investigate the issue by segmenting it into content, build, repository, configuration, deployment, runtime, and verification stages.

## Common Issues and Troubleshooting Directions

### Issue 1: Rollout succeeded, but the page is still the old version

Prioritize checking:

- The actual image of the Deployment
- Whether the Helm values were actually changed
- Whether the Service is accessing the correct Pod

### Issue 2: Pod is Running, but the business is still abnormal

Prioritize checking:

- Logs
- Configuration
- Dependent services
- Actual results of the page/API

### Issue 3: Rollback succeeded, but why are new Pods still being created?

Because rollback itself is a new update process.

### Issue 4: Which rollback method should be used for Helm vs Deployment?

Check based on your current release entry point:

- If you directly operate the Deployment, prioritize Deployment rollback
- If you manage the application through Helm, prioritize Helm rollback

---

## Key Takeaways from This Section

After completing this section, you should master:

1. The fixed order for release verification
2. Why Pod Running doesn't equate to a successful release
3. Basic operations and meaning of Deployment rollback
4. Basic operations and meaning of Helm rollback
5. How to segment troubleshoot issues based on common fault phenomena

## One-Sentence Summary

From being able to perform releases to achieving stable delivery, the key isn't executing many commands, but knowing how to verify after each release, how to roll back when issues occur, and knowing which chain to follow for fault localization.

## Summary Recommendation for This Stage

By now, the mainline of Phase 1 of the 08-CI-CD series has been fully covered.

It's recommended you don't rush to expand new tools next. Instead, first independently repeat the previous 01 to 10 sections at least once, especially:

- Build/tag/push
- kubectl deployment
- Helm deployment
- Rollout observation
- Rollback
- Fault localization

When you can perform and explain these actions independently, this CI/CD learning mainline will truly be integrated into your knowledge system.