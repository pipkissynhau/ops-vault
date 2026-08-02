# 27-Argo CD Troubleshooting Basics: Application Asynchrony, Resource Drift, and Sync Failures

## Document Notes

This is the 27th note in the 08-CI-CD learning path.

Previous articles have gradually laid out the Argo CD mainline:

- Article 19: Why Is GitOps Still Needed?
- Article 22: Minimal Installation and Access to Argo CD
- Article 23: Three Core Objects - Application, Project, Repository
- Article 25: Relationship Between Argo CD and Helm
- Article 26: Using a Git Repository to Drive a Minimal Release

This article shifts focus to:

**How to troubleshoot when Argo CD encounters issues.**

This article doesn't cover a comprehensive troubleshooting guide, but instead continues to align with your already running minimal mainline, breaking down the most common issue categories:

1. Application Not Syncing
2. Resource Drift
3. Sync Failure
4. Sync Success but Business Results Are Incorrect

And tries to write in ways that are directly observable and operable in your current environment.

This article continues to align with the current environment:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor Private Registry
- `argocd` Namespace has Argo CD installed
- Has `manual-web` Helm Chart
- Has a Minimal GitOps Repository and test Environment Application

## Tags

#Kubernetes #CI-CD #GitOps #ArgoCD #TheBarrier. #OutOfSync #SyncFailed #Helm #ResourceDrift #I'llTakeYourNotes.

## Learning Objectives

After completing this article, you should understand:

1. Why Argo CD troubleshooting needs to be layered
2. What OutOfSync, drift, and SyncFailed mean
3. Be able to reproduce a minimal drift issue in the current environment
4. Be able to locate where a sync failure might be stuck in the current environment
5. Be able to explain the recommended viewing order for Argo CD troubleshooting

## Troubleshooting Layers for This Article

At this stage, Argo CD troubleshooting is fixed into 4 layers:

1. Git / Repository Layer
2. Application Definition Layer
3. Rendering / Sync Layer
4. Cluster Operation and Business Result Layer

Once these 4 layers are clear, many issues won't be mixed together later.

---

## Part 1: Establish a Fixed Troubleshooting Order for Argo CD

In Article 21, you've already learned the troubleshooting flow for ordinary CI/CD:

- build
- push
- deploy
- verify

Now, after entering Argo CD, the troubleshooting order should be switched to a perspective more suitable for GitOps.

### Recommended Order

### Layer 1: Git / Repository Layer

First confirm:

- Is the Git repository accessible?
- Has the target state in the repository actually changed?
- Was the push operation successful?
- Does Argo CD see the correct repository content?

### Layer 2: Application Definition Layer

Then confirm:

- Is the Repository URL correct?
- Is the Path correct?
- Are the values file specifications correct?
- Is the destination namespace correct?

### Layer 3: Rendering / Sync Layer

Then confirm:

- Can the Helm Chart be normally rendered?
- Why is Argo CD not syncing?
- Where is the sync failure stuck in the process?

### Layer 4: Cluster Operation and Business Result Layer

Finally confirm:

- Are the Deployment / Pod statuses normal?
- Was the rollout successful?
- Are the page contents correct?

### Understanding to Establish at This Stage

After any Argo CD issue, don't immediately focus only on the red status in the UI.  
First ask yourself:

> Is it a Git issue, an Application configuration issue, a sync issue, or a cluster operation layer problem?

---

## Part 2: Understand the 3 Most Common Status Terms

This section must be understood first, otherwise, troubleshooting will lack foundational language.

### 1) OutOfSync

Indicates:

**The target state in Git is inconsistent with the current state in the cluster.**

It doesn't necessarily mean failure.  
Often it just means:

- Git has been changed
- The cluster hasn't synced yet

Or:

- The cluster was manually changed
- It's drifted from the Git state

### 2) Synced

Indicates:

**From Argo CD's perspective, the current cluster resource state has aligned with the Git target state.**

Note:

- Synced doesn't mean the business is absolutely fine
- It leans more toward "state alignment success"

### 3) SyncFailed

Indicates:

**Argo CD attempted to sync but failed to complete successfully.**

This usually means:

- Application definition issues
- Helm rendering issues
- Target namespace / resource object issues
- Or cluster-side apply failure

### Understanding to Establish at This Stage

The key focus of these 3 terms isn't memorizing definitions, but forming minimal judgments:

- OutOfSync: There's a discrepancy
- Synced: State has aligned
- SyncFailed: Tried to align but failed

---

## Part 3: What Does "Resource Drift (drift)" Mean?

In Article 19, you've already learned the concept of drift,  
This article turns it into something you can directly observe in your current environment.

### Drift's Most Typical Meaning

The state written in Git is one set,  
But the cluster is running another set of states.

### Typical Sources

- Manual `kubectl set image`
- Manual `kubectl edit`
- An emergency change not rewritten to Git
- A resource's labels / replicas / image, etc., key parameters were changed

### Understanding to Establish at This Stage

From a GitOps perspective, drift isn't an abstract term,  
It's:

**The cluster has deviated from the Git target state.**

---

## Part 4: Hands-on - Manually Create a Minimal Drift

This section is strongly recommended to be done, as it's the first intuition for Argo CD troubleshooting.

### Prerequisites

You've already done the following in the previous article:

- `manual-web-test` Application
- Corresponding Git Repository
- Corresponding `values-test.yaml`
- Currently in a normal sync state

### Step 1: Confirm the Current Application Status

Check in the Argo CD UI:

- Is `manual-web-test` currently `Synced`?

Or first confirm the current Deployment image via command line:

    kubectl -n test describe deploy manual-web | grep -A3 Image

### Step 2: Manually Change the Deployment to Another Tag

For example, intentionally change it to a tag not written in your Git:

    kubectl -n test set image deployment/manual-web manual-web=your Harbor domain/test/manual-web:v18

### Step 3: Refresh the Argo CD UI

Observe the status of `manual-web-test`.

### Expected Phenomenon

Most likely, it will transition from:

- `OutOfSync`

### Understanding to Be Established at This Step

You just changed the cluster,  
but Argo CD can still detect it:

- The current cluster state has diverged from the Git state

This is the minimal visualization of drift detection.

---

## Part 5: Tracing Back from OutOfSync - Should You Check Git First or the Cluster?

This step is crucial.

When you see OutOfSync, don't immediately click Sync.  
First determine:

### Case A: Did you just change Git but haven't synced yet?

For example, you just:

- Modified `values-test.yaml`
- Pushed to Git

OutOfSync is often a normal state, indicating:

- A new Git state has appeared
- The cluster hasn't caught up yet

### Case B: Did the cluster get manually changed?

Like you did just now:

- Git hasn't changed
- The cluster was manually modified

OutOfSync then resembles a "drift alert".

### Understanding to Be Established at This Step

Even though both are OutOfSync, the meaning isn't necessarily the same.  
The key is to first determine:

- Is this due to Git changes needing sync
- Or is it due to manual cluster modifications causing drift

---

## Part 6: Hands-on - Realign Drift Back to Git Target State

This section continues from the previous experiment.

### Step 1: Click Sync in the UI

Perform a sync on the currently OutOfSync Application.

### Step 2: Observe State Recovery

After syncing completes, the Application should return to:

- `Synced`

### Step 3: Check Deployment image again

    kubectl -n test describe deploy manual-web | grep -A3 Image

### Expected Phenomenon

You'll see the image reverted back to the tag defined in `values-test.yaml` in Git.

### Understanding to Be Established at This Step

At this step, you truly see:

**Argo CD doesn't just detect drift - it actively corrects it.**

This is the core value of GitOps controllers.

---

## Part 7: What is SyncFailed? Most Common Sources

This section enters real fault diagnosis.

SyncFailed generally indicates:

- Argo CD knows what the target state should be
- It tried to sync
- But failed at some step in between

The most common sources are typically 4 categories:

### 1) Repository Layer Issues

- Wrong repository address
- Wrong credentials
- Repository can't be pulled

### 2) Application Definition Layer Issues

- Wrong path
- Wrong values file name
- Wrong destination namespace

### 3) Helm Rendering Layer Issues

- Chart itself has problems
- Values overrides are incorrect
- Template rendering errors

### 4) Cluster Apply Layer Issues

- Namespace doesn't exist
- Resource conflicts
- Some resource fields are invalid
- Insufficient permissions

### Understanding to Be Established at This Step

SyncFailed isn't a single-point issue,  
It's more like saying:

- "I tried to sync, but failed at some layer"

So you must analyze layer by layer.

---

## Part 8: Hands-on - Create a Minimal SyncFailed: Wrong Values File Name

This experiment is ideal for you to try now, as it doesn't involve complex environments.

### Step 1: Open Argo CD Application Configuration

Find your current `manual-web-test` Application in the UI.

### Step 2: Intentionally change Helm values file name to a non-existent one

For example, originally it was:

- `values-test.yaml`

You intentionally change it to:

- `values-test-not-exist.yaml`

If you're managing the Application with YAML, you can directly modify the Application resource definition.

### Step 3: Refresh or re-trigger sync

### Expected Phenomenon

You'll likely see:

- Application enters an error state
- Sync fails
- Error messages will point to:
  - Values file not found
  - Helm rendering failure

### Understanding to Be Established at This Step

Such issues belong to:

**Application definition layer / Helm rendering layer problems**

Not Harbor issues, nor Deployment itself issues.

---

## Part 9: Where to Look First When SyncFailed

At this stage, it's recommended to fix this order.

### Step 1: Check Argo CD UI's sync error details

Don't just look at the "red" status,  
Instead, click into the specific error message.

Often, you'll already see:

- path not found
- values file not found
- manifest generation error
- permission denied

### Step 2: Go back to Git repository to check target state source

Verify:

- Does the file actually exist in the repository
- Is the file name correct
- Is the directory path correct

### Step 3: If it's Helm input, validate locally with `helm template`

Go to the local copy of the repository:

    cd ~/08-ci-cd/26-argocd-gitops-repo/manual-web

Run:

    helm template manual-web . -f values-test.yaml -n test

Or you'll find the corresponding error when you intentionally write it wrong.

### Understanding to Be Established at This Step

Many sync failures in Argo CD can actually be verified in reverse using the Helm tools you already know.

---

## Part 10: Hands-on - Create a Minimal SyncFailed: Wrong Target Namespace

This type is also very common.

### Step 1: Change the Application's target namespace to an incorrect or non-existent value

For example:

- `test-not-exist`

### Step 2: Resync

### Expected Phenomenon

Depending on your configuration and environment, you may see:

- Resources can't be created successfully
- Sync fails
- Or resources don't land at the expected location

### Understanding to Be Established at This Step

Such issues typically belong to:

**Application destination definition layer / cluster apply layer problems**

So when sync fails, don't just look at the Chart,  
Also check:

- Where it's being deployed to
- Whether the target namespace is reasonable

---

## Part 11: Sync Succeeded but Business Results Are Still Wrong - How to Troubleshoot

This is a situation many people are easily misled by.

You might see:

- Argo CD shows Synced
- Even Health is Healthy

But the business page is still incorrect.

### Possible Causes

1. The target state in Git is inherently wrong
2. The image.tag in values points to the wrong version
3. The content of the tag in Harbor isn't what you think
4. The page content itself was built incorrectly
5. The Service / access path isn't the resource set you think

### Current Recommended Troubleshooting Order

#### Step 1: Check Git for `values-test.yaml`

    cat ~/08-ci-cd/26-argocd-gitops-repo/manual-web/values-test.yaml

Confirm:

- What exactly is written in `image.tag`

#### Step 2: Check the actual image in Deployment

    kubectl -n test describe deploy manual-web | grep -A3 Image

#### Step 3: Verify if this tag exists in Harbor

#### Step 4: Validate page content by accessing the cluster

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh
    wget -qO- http://manual-web.test.svc.cluster.local

### Current Understanding to Establish

The meaning of `Synced` is:

- The cluster and Git target state are consistent

It does **not** mean:

- The target state in Git is necessarily the correct business version

---

## Part 12: How to Connect Argo CD Troubleshooting to the Previous Article's Mainline

This is one of the most critical understandings in this article.

Previously, you already followed this troubleshooting order:

- build
- push
- deploy
- verify

Now with Argo CD, you should add two more layers to this chain:

### Recommended Troubleshooting Flow Now:

1. Is the Git repository content correct?
2. Is the Application definition correct?
3. Did Argo CD rendering/synchronization succeed?
4. Are Deployment/Pod resources normal?
5. Is the business result correct?

### Current Understanding to Establish

Argo CD does not invalidate the previous troubleshooting chain,  
it simply makes the "configuration source" and "synchronization controller" layers more important.

---

## Part 13: Recommended Fixed Argo CD Troubleshooting Order at This Stage

It is recommended to fix the following order in the future.

### Step 1: Check Git

- Did values change correctly?
- Was the repository push successful?
- What is the actual target state?

### Step 2: Check Application

- Is the repository correct?
- Is the path correct?
- Is the values file correct?
- Is the namespace correct?

### Step 3: Check Argo CD UI status

- Is it OutOfSync or Synced?
- Is there a SyncFailed?
- What does the error detail say?

### Step 4: Optionally validate with local `helm template`

- Check if Chart rendering is normal

### Step 5: Check cluster resources

    kubectl -n test get deploy,pods,svc
    kubectl -n test describe deploy manual-web
    kubectl -n test describe pod PodName
    kubectl -n test logs PodName

### Step 6: Check business results

- Is the page content correct?

### Current Understanding to Establish

Fixing this order will make troubleshooting Argo CD issues much smoother in the future.

---

## Part 14: Practice Exercises in This Article

### Exercise 1: Manually Create a Minimal Drift

Requirements:

- The Application should be in Synced state first
- Manually change `kubectl set image` to another tag
- Observe OutOfSync
- Click Sync to revert

### Exercise 2: Manually Create a Minimal Sync Failure

Requirements:

- Intentionally write the values file name incorrectly
- Observe SyncFailed
- Check UI error details
- Restore the correct value

### Exercise 3: Answer the Following 5 Questions Yourself

1. What is the difference between OutOfSync and SyncFailed?
2. What is the most typical source of drift?
3. Why does Argo CD troubleshooting require layering?
4. Why might business results still be incorrect even if synchronization succeeds?
5. What is the biggest new layer added to Argo CD troubleshooting compared to regular CI/CD?

If you can explain these 5 questions yourself, you've mastered this article.

---

## Content to Be Able to Explain After This Article

After completing this article, it is recommended to be able to clearly explain the following:

Argo CD troubleshooting cannot only rely on whether the UI is red or green; it must first determine the problem layer—whether it's in the Git repository, Application definition, Helm rendering/synchronization, or cluster runtime layer.  
OutOfSync indicates the cluster state is inconsistent with the Git target state, which could be normal pending synchronization or resource drift; SyncFailed indicates Argo CD attempted synchronization but failed at some layer.  
Even if the state shows Synced, business validation cannot be skipped entirely, because the target state in Git might itself be incorrect.  
Therefore, the core of Argo CD troubleshooting is not learning a bunch of new commands, but adding the Git and Application layers before the existing build/push/deploy/verify troubleshooting chain.

## Common Issues and Troubleshooting Directions

### Issue 1: Why do I get anxious when I see OutOfSync?

You don't necessarily need to be anxious.  
First determine:

- Is it normal pending synchronization after Git changes?
- Or is it resource drift caused by manual cluster changes?

### Issue 2: Why does SyncFailed resemble Helm errors?

Because many synchronization failures when Argo CD integrates with Helm are essentially Chart rendering or values override issues.

### Issue 3: Why is the page still wrong after Synced?

Because Synced only means "consistent with Git",  
it does not guarantee the target state in Git is correct.

---

## Key Points Mastered in This Article

1. Minimum understanding of OutOfSync, Synced, and SyncFailed
2. Minimum experiment to reproduce drift
3. Minimum experiment to reproduce synchronization failure
4. Layered troubleshooting order for Argo CD
5. How to integrate Argo CD troubleshooting into the existing CI/CD troubleshooting chain

## One-Sentence Summary

The key to Argo CD troubleshooting is not to fixate on the synchronization button, but to first determine whether the issue lies in Git, Application, synchronization rendering, or cluster runtime and business results layer.