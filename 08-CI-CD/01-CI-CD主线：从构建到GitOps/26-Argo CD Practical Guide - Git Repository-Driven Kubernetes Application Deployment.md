# 26-Argo CD Practical: Using a Git Repository to Drive Kubernetes Application Deployment

## Document Notes

This is the 26th note in the 08-CI-CD learning path.

The previous notes have gradually laid out the Argo CD mainline:

- Note 19: Why GitOps still exists
- Note 22: Minimal Argo CD installation and access
- Note 23: Three core objects - Application, Project, Repository
- Note 25: How Argo CD connects with Helm

This note begins to truly connect the previous concepts into a minimal GitOps deployment practice:

**Use Helm Chart/values in a Git repository as the target state source, letting Argo CD drive Kubernetes application deployment.**

This note continues to align with your current existing setup without switching technologies:

- Still use `manual-web`
- Still use Helm Chart
- Still use `values-test.yaml`
- Still use `test` namespace

The current focus is not on complex platforms, but to let you see this chain in action:

Git repository configuration changes  
→ Argo CD detects the target state  
→ Argo CD syncs to K8s  
→ Business results change

This note continues to align with the current environment:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private registry
- `argocd` namespace has Argo CD installed
- Has minimal Helm Chart: `manual-web`

## Tags

#Kubernetes #CI-CD #GitOps #ArgoCD #Helm #GitRepository #DeclaredDelivery #Application #I'llTakeYourNotes.

## Learning Objectives

After completing this note, you should master:

1. Understand how a Git repository can become the target state source for Argo CD
2. Understand the minimal critical fields of Argo CD Application
3. Be able to connect the current `manual-web` Helm Chart to Argo CD
4. Be able to complete a minimal sync deployment
5. Be able to observe Argo CD-driven deployment changes by modifying values in Git
6. Be able to explain why "this is how GitOps truly lands"

## This Note's Experiment Flow

This note is divided into 5 sections:

1. Prepare a minimal Git repository as the target state source
2. Put the current Helm Chart into the Git repository
3. Create a minimal Application in Argo CD
4. Trigger a sync to complete the minimal deployment
5. Modify values in Git and observe deployment changes

---

## Part 1: First clarify what this note aims to achieve

After completing this note, you should clearly see the following:

- Not you manually `helm upgrade`
- Not you manually `kubectl set image`
- But what's written in the Git repository determines what Argo CD pulls the cluster into

This is critically different from the manual deployment and Helm manual deployment you've learned before.

### Understanding to Establish at This Step

The goal of this note isn't "to deploy another version",  
but:

**First truly run "Git as the target state source".**

---

## Part 2: Prepare a Minimal Git Repository Directory

At this stage, you can directly use the existing Helm Chart directory as a minimal Git repository experiment directory.

### Step 1: Prepare a New GitOps Experiment Directory

Execute:

    mkdir -p ~/08-ci-cd/26-argocd-gitops-repo
    cd ~/08-ci-cd/26-argocd-gitops-repo

### Step 2: Copy Current Helm Chart into the Directory

Execute:

    cp -r ~/08-ci-cd/08-helm-lab/manual-web .

### Step 3: Check the Directory

Execute:

    find manual-web -maxdepth 2 -type f | sort

You should at least see:

- `manual-web/Chart.yaml`
- `manual-web/values.yaml`
- `manual-web/values-dev.yaml`
- `manual-web/values-test.yaml`
- `manual-web/templates/deployment.yaml`
- `manual-web/templates/service.yaml`

### Understanding to Establish at This Step

At this point, you have a very typical GitOps input directory:

- Helm Chart
- Environment values
- Clear, traceable configuration structure

---

## Part 3: Initialize Local Git Repository

### Step 1: Enter the Experiment Repository Directory

    cd ~/08-ci-cd/26-argocd-gitops-repo

### Step 2: Initialize Git

    git init

### Step 3: Check Status

    git status

### Step 4: Make First Commit

    git add .
    git commit -m "init manual-web helm chart for argocd"

If your local Git hasn't been configured with username and email, set them first as prompted.

### Understanding to Establish at This Step

Now this Helm Chart is no longer just "a pile of local files",  
but begins to have a new identity:

**Git target state source.**

---

## Part 4: Prepare a Git Repository Accessible by Argo CD

This section needs to be understood in two scenarios.

### Scenario A: You currently have a Git remote repository you can push to

For example, Gitee / GitLab / GitHub any repository you can normally access and push to.  
Then push `~/08-ci-cd/26-argocd-gitops-repo` up.

### Scenario B: You currently don't have a remote repository to connect

Then this note can first walk through the steps and structure,  
and later connect to the remote repository when you have one.

But if you want Argo CD to actually connect, you still need a Git repository address it can access.

---

## Part 5: Push Local Repository to Remote Git

This section is written in the most common way, replace the remote address with your own.

### Step 1: Add Remote Repository

Execute:

    git remote add origin your Git repository address

Example:

    git remote add origin https://gitee.com/Your username/argocd-manual-web.git

### Step 2: Push to Remote

If the current main branch is called master:

    git push -u origin master

If it's main:

    git push -u origin main

### Step 3: Confirm on Remote Repository Page

Confirm you can see:

- `manual-web/Chart.yaml`
- `manual-web/values-test.yaml`
- `manual-web/templates/`

### Understanding to Establish at This Step

After this step, you truly have:

**A Git target state source accessible by Argo CD.**

---

## Part 6: First Confirm Current Target State in `values-test.yaml`

Check in the local repository directory: /think

cd ~/08-ci-cd/26-argocd-gitops-repo
cat manual-web/values-test.yaml

You must at least confirm these fields:

- `image.repository`
- `image.tag`
- `replicaCount`
- `namespace`

For example your current configuration might be:

    replicaCount: 2

    image:
      repository: your Harbor domain/test/manual-web
      tag: "test-13"
      pullPolicy: IfNotPresent

    namespace: test

### Understanding at this step

This file is one of the core target states for the test environment later.  
Argo CD doesn't guess what the test environment should be - it reads what's written here.

---

## Part 7: Adding a Repository to Argo CD

This section begins the Argo CD setup.

### Step 1: Confirm Argo CD page is accessible

If the previous port-forward has ended, re-run:

    kubectl -n argocd port-forward svc/argocd-server 8080:443

Open browser to:

    https://127.0.0.1:8080

### Step 2: Log in to Argo CD

The username is typically:

    admin

If you've forgotten the password, you can retrieve it:

    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo

### Step 3: Add Git Repository

In the Argo CD UI, find:

- Settings
- Repositories
- Add repository

Enter your remote repository address.

#### For public repositories

Usually just fill in the URL.

#### For private repositories

You need additional configuration:

- Username / Password
- Or Token
- Or SSH key

### Understanding at this step

The Repository object solves:

**Where Argo CD gets its configuration from.**

---

## Part 8: Creating a Minimal test Environment Application

This is the core action of this article.

### Step 1: Enter Applications Page

Click:

- Applications
- New App

### Step 2: Fill in Key Fields

Currently you should focus on these fields.

#### 1) Application Name

For example:

    manual-web-test

#### 2) Project

Use the default first:

    default

We won't expand on complex Project rules at this stage.

#### 3) Sync Policy

Currently recommend starting with manual sync to better observe the process.  
Understand automatic sync later.

#### 4) Repository URL

Select the Git repository you just added.

#### 5) Path

Enter the Chart path, for example:

    manual-web

#### 6) Cluster URL / Destination Cluster

Usually the current cluster is:

    https://kubernetes.default.svc

If the UI defaults to in-cluster, just select the current cluster.

#### 7) Namespace

Enter:

    test

### Step 3: Declare this is a Helm type input

In the Source section, you need to tell Argo CD:

- This is a Helm Chart path

And specify:

- `values-test.yaml`

Different UI versions may display this differently, but there is usually a Helm-related area where you can enter:

- Value files
- Parameters

Currently focus on letting it use:

    values-test.yaml

### Understanding at this step

The Application here truly expresses:

- From which repository
- Which path
- Which Helm values
- To which namespace

This is a complete GitOps deployment object.

---

## Part 9: After creating Application, check status first, don't sync too quickly

### Step 1: Observe Application page after creation

You'll usually see key information like:

- Sync Status
- Health Status

### Possible initial statuses

If no sync has occurred yet, it might look like:

- OutOfSync
- Missing
- Unknown / Progressing

This is normal.

### Understanding at this step

Application creation success doesn't mean resources have been deployed to the cluster.  
It just establishes the "target state relationship" first.

---

## Part 10: Execute the first sync

### Step 1: Click Sync on Application page

If using manual sync policy, click:

- Sync

### Step 2: Observe sync process

After sync completes, check:

- Sync Status
- Health Status

### Step 3: Check cluster resources from command line

Run:

    kubectl -n test get deploy,pods,svc

If you previously had same-named resources in test environment, pay special attention to whether current actual state has been taken over by this target state.

### Understanding at this step

This moment is critical because you'll see:

- Not manual `kubectl apply`
- Not manual `helm upgrade`
- But the Application sync action has deployed the target state to the cluster

This is the beginning of GitOps controller behavior.

---

## Part 11: Validate business results

### Step 1: Confirm Deployment image

Run:

    kubectl -n test describe deploy manual-web | grep -A3 Image

### Step 2: Validate page content

Run:

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

After entering, run:

    wget -qO- http://manual-web.test.svc.cluster.local

### Expected phenomena

You should see content corresponding to the tag of `values-test.yaml` currently, for example:

    version: test-13

### Understanding at this step

Here you need to truly connect two things:

- What tag is written in Git for `values-test.yaml`
- Whether the cluster is currently running the same tag

If consistent, this is GitOps working.

---

## Part 12: Do a minimal GitOps change experiment

This section is one of the most valuable parts of the entire document.

### Step 1: Prepare a New test Image

Enter the application directory:

    cd ~/08-ci-cd/01-manual-release

Modify the content:

    cat > index.html <<'EOF'
    <html>
      <head>
        <meta charset="utf-8">
        <title>manual release test-26</title>
      </head>
      <body>
        <h1>Hello K8s CI/CD</h1>
        <p>version: test-26</p>
      </body>
    </html>
    EOF

Build and push:

    docker build -t manual-web:test-26 .
    docker tag manual-web:test-26 Yours.HarborDomain Name/test/manual-web:test-26
    docker push Yours.HarborDomain Name/test/manual-web:test-26

### Step 2: Return to the GitOps Repository, Modify `values-test.yaml`

Enter:

    cd ~/08-ci-cd/26-argocd-gitops-repo

Change:

    tag: "Original Value"

To:

    tag: "test-26"

### Step 3: Commit and Push Git Changes

Execute:

    git add manual-web/values-test.yaml
    git commit -m "update test environment to test-26"
    git push

### Current Understanding to Establish at This Step

This time, you are not directly modifying the Deployment,  
nor directly `helm upgrade`,  
but instead:

**Only modify the target state in Git.**

This is the key action of GitOps.

---

## Part 13: Return to Argo CD, Observe State Changes

### Step 1: Refresh the Application Page

You will likely see:

- The Application status has changed
- It may show OutOfSync

### Why This Happens

Because:

- The target state in Git has changed
- The current state of the cluster has not changed
- Argo CD detects the inconsistency

### Current Understanding to Establish at This Step

This is the visible realization of the "state comparison" mentioned in the 19th article.

Argo CD does not blindly synchronize; it first checks:

- What is in Git
- What is the current state of the cluster
- Whether they are consistent

---

## Part 14: Execute Synchronization, Let Argo CD Complete the Release

### Step 1: Perform Sync Again via UI

Click:

- Sync

### Step 2: Observe State Recovery

After successful synchronization, it will typically return to:

- Synced
- Healthy

### Step 3: Verify Cluster Resources

Execute:

    kubectl -n test describe deploy manual-web | grep -A3 Image

You should see:

    test-26

### Step 4: Revalidate Page Content

Execute:

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

After entering, execute:

    wget -qO- http://manual-web.test.svc.cluster.local

### Expected Phenomenon

You should see:

    version: test-26

### Current Understanding to Establish at This Step

At this point, you have truly completed a minimal GitOps release:

- Git was modified
- Argo CD detected the inconsistency
- Argo CD synchronized
- Cluster state changed
- Business results changed

---

## Part 15: What Is the Difference Between This Experiment and Your Previous Manual Helm Release

This is very important; make sure you can explain it clearly.

### Previous Manual Helm Release

You did the following:

1. Modify `values-test.yaml`
2. Manually execute:
       helm upgrade ...

### This GitOps Release

You did the following:

1. Modify `values-test.yaml`
2. Commit to Git
3. Argo CD detected inconsistency between Git and the cluster
4. Argo CD synchronized

### Core Difference

#### Manual Helm Mode

"I actively issue a release command"

#### Argo CD + Helm Mode

"I modify the target state in Git, and the controller handles alignment"

### Current Understanding to Establish at This Step

This is the boundary between GitOps and traditional manual releases after true implementation.

---

## Part 16: Complex Points to Avoid at This Stage

To maintain the pace, this section will not expand on these complex topics:

- Multi-cluster Application
- Details of automatic sync strategies
- Self-healing (Self-Heal)
- Multi-environment Application orchestration
- App of Apps
- Fine-grained resource difference ignore strategies

These will be covered in dedicated topics later.

### Most Important at This Stage

1. Truly run through the minimal chain of Git → Argo CD → K8s
2. Feel that "modifying Git is modifying the target state"
3. Feel Argo CD's synchronization behavior

---

## Part 17: This Section's Mini Exercise

### Exercise 1: Re-do a Minimal GitOps Release Yourself

Requirements:

- Prepare a new image tag, e.g., `test-26b`
- Push to Harbor
- Modify the `values-test.yaml` in the Git repository
- Commit + Push
- Return to Argo CD and observe OutOfSync
- Sync again and verify the page results

### Exercise 2: Draft a dev Environment Application Idea

You don't need to create it yet, but you must clearly write:

- Name:`manual-web-dev`
- Path:`manual-web`
- Values:`values-dev.yaml`
- Namespace:`dev`

### Exercise 3: Answer the Following 5 Questions Yourself

1. What role does the Git repository play in this chain?
2. What are the minimal critical fields of an Argo CD Application?
3. Why is modifying Git more aligned with GitOps than directly modifying the Deployment?
4. When Argo CD connects to Helm, which layer does it truly connect to?
5. What is the core difference between this experiment and the previous manual `helm upgrade`?

If you can explain these 5 questions yourself, you've mastered this section.

---

## Content to Be Able to Explain After Completing This Section

After completing this section, it's recommended that you can clearly explain the following statement:

# Argo CD Practical Application Key

The key to Argo CD practical application is not to learn many buttons first, but to truly make the Git repository the source of the desired state.  
In this experiment, the Git repository stores `manual-web` Helm Chart and `values-test.yaml`. The Argo CD Application is responsible for declaring: from which repository, which path, using which values, and synchronizing the desired state to which namespace.  
When the values in Git change, Argo CD will detect that the cluster state and desired state are inconsistent, and synchronize the cluster to the new desired version through synchronization.  
This differs from the previous manual `helm upgrade` in the biggest way: the change entry is no longer the command line, but configuration changes in Git.

## Common Issues and Troubleshooting Directions

### Issue 1: Application is created but never synchronizes

First check:

- Is the repository connected successfully?
- Is the path written correctly?
- Is the values file name written correctly?
- Does the target namespace exist?
- Can the Chart be rendered normally?

### Issue 2: Cluster remains unchanged after synchronization

First check:

- Is the Git push really successful?
- Is the repo revision currently seen by the Application the latest?
- Has the tag of `values-test.yaml` really changed?
- Does the tag exist in Harbor?

### Issue 3: Why do I feel like it's just replacing the Helm upgrade entry?

This indicates you've grasped the key.  
The difference lies in:

- Manual Helm: You issue a command
- Argo CD + Helm: You modify Git, and the controller triggers a synchronization action

---

## Key Points to Master in This Article

1. How a Git repository becomes the source of the desired state for Argo CD
2. The minimal critical fields of an Application
3. The minimal practical workflow of Argo CD + Helm
4. How Git changes drive cluster deployment
5. The core difference between GitOps and manual Helm deployment

## One-Sentence Summary

The essence of Argo CD practical application is not to run Helm again, but to make the Chart and values in Git the sole source of the desired state, and let Argo CD continuously synchronize this desired state to the Kubernetes cluster.