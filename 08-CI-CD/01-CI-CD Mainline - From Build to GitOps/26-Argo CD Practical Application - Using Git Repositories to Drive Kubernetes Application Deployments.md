# 26-Argo CD Practical Application: Using Git Repositories to Drive Kubernetes Application Deployments

## Document Description

This article is the 26th note in the 08-CI-CD learning pathway.

Previous articles have gradually introduced the main concepts of Argo CD:

- Article 19: Why GitOps Still Matters
- Article 22: Minimum Installation and Access for Argo CD
- Article 23: The Three Core Objects of Application, Project, and Repository
- Article 25: How Argo CD Integrates with Helm

In this article, we will put these concepts into practice to perform a minimal GitOps deployment:

**Use the Helm Chart/values in the Git repository as the target state source, and let Argo CD drive the Kubernetes application deployment.**

This article will continue to use the technologies you already have, without changing them:

- Still use `manual-web`
- Still use Helm Chart
- Still use `values-test.yaml`
- Still use the `test` namespace

The focus here is not on building complex systems but on demonstrating how this entire chain works:

Changes in the Git repository configuration  
→ Argo CD recognizes the target state  
→ Argo CD synchronizes it with K8s  
→ The business results change accordingly

This article will also use the current environment:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private repository
- The `argocd` namespace already has Argo CD installed
- We already have a minimal Helm Chart: `manual-web`

## Tags

#Kubernetes #CI-CD #GitOps #ArgoCD #Helm #Git Repository #Declarative Delivery #Application #Practical Notes

## Learning Objectives

After completing this article, you should be able to:

1. Understand how the Git repository can serve as the target state source for Argo CD
2. Know the minimum critical fields of an Argo CD Application
3. Integrate the current `manual-web` Helm Chart into Argo CD
4. Perform a minimal synchronization and deployment
5. Observe changes in the deployment by modifying the values in Git
6. Clearly explain why this represents the true implementation of GitOps

## Main Experimental Steps of This Article

This article is divided into 5 sections:

1. First, prepare a minimal Git repository as the target state source.
2. Place the current Helm Chart in the Git repository.
3. Create a minimal Application in Argo CD.
4. Trigger a synchronization to complete the minimum deployment.
5. Modify the values in Git and observe the resulting changes in the deployment.

---

## Section 1: Determine What This Article aims to Achieve

After completing this article, you should be able to clearly see the following:

- It's not about manually executing `helm upgrade` or `kubectl set image`.
- Instead, Argo CD will adjust the cluster according to the contents of the Git repository.

This is a crucial difference from the manual deployment and Helm manual configuration methods you learned before.

### Understanding Required for This Step

The goal of this article is not to “release another version” but to:

**For the first time, actually implement ‘using Git as the target state source.’**

---

## Section 2: Prepare a Minimal Git Repository Directory

At this stage, you can directly use the directory where your current Helm Chart is stored as the experimental Git repository directory.

### Step 1: Prepare a New GitOps Experimental Directory

Execute:

    mkdir -p ~/08-ci-cd/26-argocd-gitops-repo
    cd ~/08-ci-cd/26-argocd-gitops-repo

### Step 2: Copy the Current Helm Chart into It

Execute:

    cp -r ~/08-ci-cd/08-helm-lab/manual-web .

### Step 3: Check the Directory Contents

Execute:

    find manual-web -maxdepth 2 -type f | sort

You should see at least the following files:

- `manual-web/Chart.yaml`
- `manual-web/values.yaml`
- `manual-web/values-dev.yaml`
- `manual-web/values-test.yaml`
- `manual-web/templates/deployment.yaml`
- `manual-web/templates/service.yaml`

### Understanding Required for This Step

By now, you have a typical GitOps input directory:

- Helm Chart
- Environment values
- A clearly traceable configuration structure

---

## Section 3: Initialize the Local Git Repository

### Step 1: Enter the Experimental Repository Directory

    cd ~/08-ci-cd/26-argocd-gitops-repo

### Step 2: Initialize Git

    git init

### Step 3: Check the Status

    git status

### Step 4: Make Your First Commit

    git add .
    git commit -m "init manual-web helm chart for argocdThe following content will explain automatic synchronization in detail.

#### 4) Repository URL

Select the Git repository you just added.

#### 5) Path

Enter the path where the Chart is located, for example:

    manual-web

#### 6) Cluster URL / Destination Cluster

Usually, the current cluster is:

    https://kubernetes.default.svc

If the UI indicates that it is the default in-cluster cluster, simply select the current cluster.

#### 7) Namespace

Enter:

    test

### Step 3: Declare This as a Helm Type Input

In the Source section, you need to tell Argo CD:

- That this is the path to the Helm Chart

And specify:

- `values-test.yaml`

The display may vary depending on the UI version, but there is usually a Helm-related area where you can enter:

- Value files
- Parameters

The key at this stage is to ensure it uses:

    values-test.yaml

### Understanding Required for This Step

Here, "Application" actually refers to:

- Which repository to use
- Which path to follow
- Which Helm values file to apply
- Which namespace to deploy to

This constitutes a complete GitOps deployment object.

---

## Section 9: After Creating an Application, Check the Status First—Don’t Rush

### Step 1: Observe the Application Page After Creation

You will typically see some key information, such as:

- Sync Status
- Health Status

### Possible Initial States

If synchronization has not yet occurred, you may see:

- OutOfSync
- Missing
- Unknown / Progressing

This is completely normal.

### Understanding Required for This Step

Just because the Application has been successfully created does not mean that the resources have been deployed to the cluster.  
It merely establishes the "target state relationship" first.

---

## Section 10: Perform the First Synchronization

### Step 1: Click on Sync on the Application Page

If it is a manual synchronization strategy, click now:

- Sync

### Step 2: Observe the Synchronization Process

After synchronization is complete, check again:

- Sync Status
- Health Status

### Step 3: Return to the Command Line to Check Cluster Resources

Execute:

    kubectl -n test get deploy,pods,svc

If there are already resources with the same name in the `test` environment before, pay special attention to whether their current state has been replaced by the target state set by GitOps.

### Understanding Required for This Step

This is a crucial moment because you will see:

- It is not a manual `kubectl apply`
- It is not a manual `helm upgrade`
- Instead, it is the Application’s synchronization action that applies the target state to the cluster

This marks the beginning of the GitOps controller’s functionality.

---

## Section 11: Verify the Business Results

### Step 1: Confirm the Image in the Deployment

Execute:

    kubectl -n test describe deploy manual-web | grep -A3 Image

### Step 2: Verify the Page Content

Execute:

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

After entering, execute:

    wget -qO- http://manual-web.test.svc.cluster.local

### Expected Result

You should see the version content corresponding to the current tag in `values-test.yaml`, for example:

    version: test-13

### Understanding Required for This Step

Here, you need to ensure that two things are consistent:

- What tag is specified in `values-test.yaml` in Git
- Which tag is currently running in the cluster

If they match, it means GitOps is working correctly.

---

## Section 12: Conduct a Simple GitOps Change Experiment

This section contains one of the most valuable parts of the entire guide.

### Step 1: Prepare a New `test` Image

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

Build and push it:

    docker build -t manual-web:test-26 .
    docker tag manual-web:test-26 your-Harbor-domain/test/manual-web:test-26
    docker push your-Harbor-domain/test/manual-web:test-26

### Step 2: Return to the GitOps Repository and Modify `values-test.yaml`

Enter:

    cd ~/08-ci-cd/26-argocd-gitops-repo

Change:

    tag: "original value"

to:

   4. When Argo CD connects to Helm, which layer is actually being connected to?
5. What is the core difference between this experiment and the previous manual `helm upgrade`?

If you can answer these 5 questions yourself, then you have mastered this topic.

---

## Key Points You Should Be Able to Explain After This Chapter

After completing this chapter, it is recommended that you can clearly explain the following:

The key to practical use of Argo CD is not about learning many commands first, but rather about turning the Git repository into the true source of the target state.  
In this experiment, the Git repository contains the `manual-web` Helm Chart and `values-test.yaml`. The Argo CD Application is responsible for specifying which repository, path, and values to use in order to synchronize the target state to a particular namespace.  
When changes are made to the values in Git, Argo CD will detect that the cluster status is inconsistent with the target state and will update the cluster to the new target version through synchronization.  
The main difference between this approach and the previous manual `helm upgrade` is that the point of entry for making changes is no longer the command line, but rather the configuration changes in Git.

## Common Issues and Troubleshooting Directions

### Issue 1: The Application has been created, but there is no synchronization

First, check:
- Whether the repository has been successfully connected.
- Whether the path is correct.
- Whether the name of the values file is correct.
- Whether the target namespace exists.
- Whether the Chart can be rendered correctly.

### Issue 2: No changes occur in the cluster after synchronization

First, check:
- Whether the Git push was successful.
- Whether the current repo revision seen by the Application is the latest.
- Whether the tag in `values-test.yaml` has actually been changed.
- Whether the tag exists in Harbor.

### Issue 3: Why does it feel like I am just replacing `helm upgrade` with another method?

This indicates that you have grasped the key difference.  
The difference lies in:
- With manual Helm: You issue commands.
- With Argo CD + Helm: You make changes to Git, and the controller handles the synchronization.

---

## Key Takeaways of This Chapter

1. How a Git repository can become the target state source for Argo CD.
2. The minimum critical fields required for an Application in Argo CD.
3. The basic practical workflow of using Argo CD with Helm.
4. How changes in Git drive cluster updates.
5. The core difference between GitOps and manual Helm deployment methods.

## In One Sentence

The essence of practical use of Argo CD is not about executing Helm commands again, but rather about using the Chart and values in Git as the sole source of the target state, with Argo CD continuously synchronizing this state to the Kubernetes cluster.