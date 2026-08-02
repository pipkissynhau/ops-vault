# 19-GitOps Basics: Why Argo CD Still Exists

## Documentation Notes

This article is the 19th note in the 08-CI-CD learning path.

In the previous 01-18, you've already established a minimal deployment pipeline and gradually completed:

- Manual deployment
- Image building and tagging
- Harbor repository
- GitLab CI / Jenkins Pipeline
- K8s rolling updates
- Helm
- Multi-environment deployment
- Executor
- CI/CD security
- Harbor advanced governance

In this article, we begin to enter a very critical expansion topic:

**GitOps.**

This article doesn't require you to immediately use Argo CD or set up the entire GitOps platform today.  
The current focus is to clearly explain the following question:

> We already have GitLab CI, Jenkins, kubectl, and Helm for deployment, why do we still need Argo CD and GitOps later?

If this question isn't clear, you'll feel Argo CD is like "reinventing the wheel".  
Actually, it's not.

This article continues to align with your current learning path and experimental environment:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private repository
- `test` Namespace
- Completed minimal application `manual-web` build / push / deploy / rollback / Helm deployment experiments

## Tags

#Kubernetes #CI-CD #GitOps #ArgoCD #Helm #kubectl #Declaration #ConfigurationManagement #I'llTakeYourNotes.

## Learning Objectives

After completing this article, you should understand:

1. What problems GitOps aims to solve
2. The boundary between GitOps and traditional CI/CD
3. Why "being able to deploy" still requires "continuous alignment with target state"
4. Where Argo CD fits in the entire pipeline
5. Compare and understand the value of GitOps using the deployment methods you've already learned
6. Clearly explain "why Argo CD still exists"

## This Article's Experimental Focus

This article doesn't install Argo CD first. Instead, we'll do 3 things first:

1. Identify the boundaries of the kubectl / Helm deployment methods you already know
2. Understand the core idea of GitOps: Git as the target state
3. Simulate a "Git target state" approach using the current experiment directory

---

## Part 1: First, Review the Deployment Methods You Already Know

You've already done two of the most common deployment methods.

### Method 1: Direct kubectl Deployment

You executed:

    kubectl -n test set image deployment/manual-web manual-web=your Harbor domain/test/manual-web:v10

Or:

    kubectl apply -f manual-web.yaml

### Method 2: Deployment via Helm

You executed:

    helm upgrade manual-web . -n test

Or with values:

    helm upgrade manual-web-test . -f values-test.yaml -n test

### Understanding to Establish Now

You now have the ability to "deploy".  
This is important because GitOps isn't replacing deployment foundations, but building on this foundation to solve new problems.

---

## Part 2: First, Look at the Commonalities of These Deployment Methods

Whether you use kubectl or Helm, they share a common characteristic:

**The deployment action is initiated by you actively.**

That is:

- You execute a command
- K8s receives the command
- Cluster state changes

From an action perspective, this model is:

> **Push-type deployment**

That is:

- People or pipelines push changes into the cluster

### Understanding to Establish Now

The previous deployment methods are correct and very common.  
But their core characteristic is:

- Changes come from external active pushes
- The cluster doesn't actively align with a "declared target state"

This is the background for GitOps to emerge later.

---

## Part 3: Several Common Issues with Current Methods

This section is very critical because GitOps emerged precisely based on these issues.

### Issue 1: Cluster Actual State May Differ from Your Perception

For example:

- You think the Deployment should be `v10`
- But someone manually changed it to `v9`
- Or a temporary fix directly modified resources in the cluster
- Resulting in inconsistency between Git / local files / actual cluster state

### Issue 2: It's Not Always Clear Who Changed What in Git

If many operations are directly:

- `kubectl edit`
- `kubectl set image`
- Manual `helm upgrade`

Can you clearly answer:

- Where did the current state come from in Git?
- Who made the change?
- Why was it changed this way?

Not necessarily.

### Issue 3: Cluster Drift (Drift) Is Easy to Occur

So-called drift, is:

- What you write in Git or templates is one state
- But the cluster actually runs another state

And this deviation won't automatically recover.

### Issue 4: State Tracking Becomes Harder in Multiple Environments

Once you have:

- dev
- test
- prod

Combined with:

- Helm values
- Multiple version tags
- Multiple namespaces

Without a unified "target state source", environment states become harder to track.

### Understanding to Establish Now

GitOps isn't because previous kubectl / Helm "aren't good enough",  
but because:

**When environments grow and collaboration becomes more complex, relying solely on external push deployments is unstable and hard to track.**

---

## Part 4: First Understand GitOps with One Sentence

At this stage, just remember the most important sentence:

> **The core idea of GitOps is to treat the configuration in the Git repository as the target state the cluster should align with long-term.**

This sentence has two key terms:

### 1) Git as the Target State Source

Not "reference", but:

- What's written in Git
- The cluster should try to become that

### 2) Long-term Alignment

Not just a one-time deployment,  
but if the cluster deviates from the Git state, it should be pulled back.

### Understanding to Establish Now

GitOps isn't just "using Git to manage configurations",  
but:

**Making Git the factual source of cluster state.**

---

## Part 5: What Does "Target State" Mean

This point must be thoroughly explained.

Assume you have a configuration in Git that specifies:

- Deployment's image should be `manual-web:v19`
- replicas should be 2
- Service port should be 80

This configuration describes:

**The state you want the cluster to eventually become.**

This is called the target state.

### Difference from Previous Methods

Previously, you executed:

    kubectl set image ...

Which is more like saying: /think

- "I'm going to send a modification command now"

GitOps is more like saying:

- "The cluster should always remain in the state defined in Git"

### Understanding at this stage

The focus of GitOps is not just "triggering a single release",  
but rather:

**Continuously ensuring the cluster approaches and maintains the target state.**

---

## Part 6: Understanding the boundary between GitOps and traditional CI/CD

This question is particularly important.

Many people mistakenly believe:

- With GitOps, you don't need CI/CD anymore

This is inaccurate.

### A more reasonable understanding at this stage

#### Traditional CI is more focused on the first half

Responsible for:

- Pulling code
- Testing
- Building
- Tagging
- Pushing to Harbor

That is:

**Transforming "application content" into "image artifacts"**

#### GitOps is more focused on the latter half

Responsible for:

- Keeping cluster configuration aligned with the target state in Git
- Detecting drift
- Triggering synchronization
- Returning Deployment/Service/Helm Release to the state declared in Git

That is:

**Continuously applying the "configuration target" to the cluster**

### Understanding at this stage

So a more complete workflow is typically:

Code  
→ CI builds image  
→ Push to Harbor  
→ Update deployment configuration in Git  
→ GitOps tool (like Argo CD) aligns the cluster with this Git state

This is why the previous knowledge you learned won't be wasted.

---

## Part 7: Why Argo CD isn't just reinventing the wheel

You've already learned:

- `kubectl apply`
- `kubectl set image`
- `helm upgrade`

So why Argo CD?

Because Argo CD solves problems that aren't "how to send commands",  
but rather:

- Who continuously monitors the difference between Git and the cluster
- Who pulls the drifted state back
- Who truly implements the idea that "Git is the source of the target state"

### You can understand it this way

#### kubectl/helm are more like

- Tools you actively send commands with

#### Argo CD is more like

- A controller that constantly monitors whether Git state and cluster state are aligned

### Understanding at this stage

Argo CD isn't meant to replace all the commands you learned earlier,  
but rather to turn:

**"What the configuration should be"**  
into a continuously aligned system behavior.

---

## Part 8: Simulate "Git target state" using your current experiment directory

This section is very important because it transforms GitOps from an abstract concept into something you can understand.

### Step 1: Return to the Helm Chart directory

Execute:

    cd ~/08-ci-cd/08-helm-lab/manual-web

This is already very close to the "configuration source" in GitOps.

Because here you have:

- `Chart.yaml`
- `templates/`
- `values.yaml`
- `values-dev.yaml`
- `values-test.yaml`

### Step 2: Consider this directory as the "target state directory" in a Git repository

For example:

- `values-test.yaml` contains the target state for the `test` environment
- `values-dev.yaml` contains the target state for the `dev` environment

At this point you can imagine:

If this directory were placed in a Git repository,  
and there was a system that constantly monitored:

- What `values-test.yaml` writes in Git is `test-13`
- Whether manual-web-test in the cluster is also `test-13`

That "monitoring alignment" system is exactly what a GitOps controller does.

### Understanding at this stage

You actually already have the core half of GitOps:

- Target state configuration files

You just don't have the "continuously monitoring and synchronizing" controller yet.

---

## Part 9: Do a "manual simulation of GitOps alignment" experiment

This section is very critical.

We can manually simulate a GitOps mindset without using Argo CD.

### Step 1: Confirm the current version in the test environment

Execute:

    kubectl -n test describe deploy manual-web | grep -A3 Image

Note down the current actual image tag.

### Step 2: Check the target state for the test environment in Helm

Execute:

    cd ~/08-ci-cd/08-helm-lab/manual-web
    cat values-test.yaml

Look at the following:

- `image.tag`
- `replicaCount`
- `namespace`

### Step 3: Intentionally make the cluster inconsistent with values-test.yaml

For example, currently `values-test.yaml` contains:

    tag: "test-13"

You can manually change the Deployment to another tag, like:

    kubectl -n test set image deployment/manual-web manual-web=your-Harbor-domain/test/manual-web:v18

### Step 4: Check the differences again

Now:

- Git/Chart target state: still `test-13`
- Cluster actual state: changed to `v18`

This is the smallest example of "state drift".

### Step 5: Manually re-run Helm upgrade to bring the state back

Execute:

    helm upgrade manual-web-test . -f values-test.yaml -n test

Then check again:

    kubectl -n test describe deploy manual-web | grep -A3 Image

### Understanding at this stage

The actions you just manually completed essentially are:

- Detecting the cluster deviating from the target state
- Manually bringing it back to the target state

What GitOps/Argo CD aims to do is automate and continuously perform this process.

---

## Part 10: What is "drift (drift)"

At this point, explaining drift becomes very natural.

Drift is:

**The current real state of the cluster deviates from the target state declared in Git.**

### Typical sources

- Someone manually changed the Deployment
- An emergency fix was directly performed on the cluster
- After a rollout, it wasn't synchronized back to Git
- Temporary resource parameter changes in the environment

### Why drift is dangerous

Because once drift occurs:

- Git no longer represents the real environment
- The environment becomes increasingly difficult to manage
- When troubleshooting, people have different understandings of the "current state"

### Understanding at this stage

One of the values of GitOps is to make drift easier to detect, and even automatically correct it.

---

## Part 11: Why GitOps is particularly suitable for multi-environment scenarios

You've already learned in the 13th and 16th articles: /think

- Differences between dev, test, and prod environments
- Overriding strategy for values-dev.yaml / values-test.yaml

GitOps works particularly well here because:

### Each environment can have its own target state file

For example:

- `values-dev.yaml`
- `values-test.yaml`
- `values-prod.yaml`

### Each environment can have its own "expected appearance"

A GitOps controller just needs to know:

- Which configuration dev should align with
- Which configuration test should align with
- Which configuration prod should align with

to manage multiple environments more clearly.

### Current understanding to establish at this stage

GitOps and Helm values work very well together,  
because Helm values naturally suit expressing "target states for different environments".

---

## Part 12: Understanding Argo CD's role at this stage

Now looking back at Argo CD, it will make more sense.

You can first understand it as:

**A controller specifically monitoring whether Git state and Kubernetes state are consistent.**

It typically does these things:

1. Inspect configuration in Git repositories
2. Inspect current cluster resources
3. Compare differences between the two
4. Decide whether to synchronize
5. Pull cluster state to match target state

### Current understanding to establish at this stage

Argo CD is not replacing you to build images,  
nor is it replacing you to write Dockerfiles,  
but rather it is more inclined to:

**A continuous aligner for cluster configuration states.**

---

## Part 13: Recommended understanding of GitOps and Argo CD at this stage

Suggested to first fix this minimal perspective.

### GitOps

Is a philosophy of release and configuration management:

- Git is the source of target state
- Cluster should continuously align with Git state

### Argo CD

Is a common implementation tool for this philosophy:

- Inspect Git
- Inspect cluster
- Perform synchronization
- Handle drift

### Current understanding to establish at this stage

Do not understand GitOps as "a single software".  
It first is a philosophy, Argo CD is one of the most typical implementations.

---

## Part 14: Most suitable boundary understanding to establish at this stage

This part is very important.

### Do not misunderstand at this stage

- Having GitOps means you don't need CI anymore
- Having Argo CD means you don't need Harbor anymore
- Having Argo CD means you don't need Helm anymore

These are all incorrect.

### More reasonable understanding is

- CI: Responsible for building, testing, pushing images
- Harbor: Responsible for storing images
- Helm: Responsible for organizing configuration templates
- Git: Responsible for saving target state
- Argo CD: Responsible for continuously aligning cluster state with Git target state

### Current understanding to establish at this stage

GitOps is not replacing all the previous things,  
but rather re-connecting the capabilities already learned into a more stable release model for the latter half of the process.

---

## Part 15: This article's practice exercise

### Practice 1: Simulate a "state drift"

Requirements:

1. First confirm `values-test.yaml`'s `image.tag`
2. Then manually use `kubectl set image` to change Deployment to another tag
3. Compare "target state" and "current state"
4. Then execute `helm upgrade` to pull it back

### Practice 2: Draw GitOps's position in the current main line

Requirements to draw at least:

- Code
- Harbor
- Helm / values
- Git
- K8s cluster
- Argo CD (first as a conceptual position)

### Practice 3: Answer the following 5 questions yourself

1. What problem does GitOps aim to solve?
2. What is a target state?
3. What is state drift?
4. What is the main difference between Argo CD and kubectl / Helm?
5. Why is GitOps not replacing CI, but rather having different responsibilities?

If you can explain these 5 questions yourself, you've mastered this article.

---

## Content to be able to explain after this article

After completing this article, it's recommended to be able to explain the following:

Previously, through kubectl or Helm release methods, essentially it's people or pipelines actively pushing changes into the cluster.  
This method, when environments become more complex and collaboration becomes more intricate, easily leads to inconsistencies between Git configuration and the actual cluster state, which is called state drift.  
The core idea of GitOps is to treat the configuration in the Git repository as the target state that the cluster should maintain long-term, while tools like Argo CD's role is to continuously compare Git state and cluster state, and synchronize when there are differences.  
Therefore, GitOps is not replacing CI, Harbor, and Helm, but rather building on these capabilities to make the configuration alignment of the cluster's latter half more clear and stable.

## Common Questions and Troubleshooting Directions

### Question 1: I already know Helm, why do I need GitOps?

Because Helm solves template-based and parameterized releases,  
but GitOps further solves:

- Continuous alignment of target state
- Drift detection
- Multi-environment state tracking

### Question 2: Is GitOps mandatory to have all environments fully set up before learning?

No.  
At this stage, establishing the core concepts of "target state" and "drift" is more important.

### Question 3: Will I completely stop using kubectl in the future?

No.  
kubectl remains a fundamental tool, and GitOps simply shifts the main entry point for "daily formal releases and state alignment" to Git/controller.

---

## Key Points of This Article

1. The problem GitOps aims to solve
2. The meaning of Git as a target state source
3. The concept of state drift
4. Argo CD's position in the main line
5. The boundary between GitOps and traditional CI/CD

## One-Sentence Summary

The core of GitOps is not "adding another release tool," but making Git the source of the cluster's configuration target state, and using controllers like Argo CD to continuously align the actual cluster state with the declared Git state.