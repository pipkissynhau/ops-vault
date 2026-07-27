# 25-Integrating Argo CD with Helm: From Charts to GitOps Deployment

## Documentation Overview

This document is the 25th installment of the 08-CI-CD learning pathway.

Previous articles have gradually laid out the foundational concepts of Argo CD:

- Article 19: Why GitOps and Argo CD Exist
- Article 22: Basic Installation and Initial Setup of Argo CD
- Article 23: The Three Core Components: Application, Project, and Repository

You have also previously learned about Helm:

- Charts, templates, and values
- Commands like `helm install`, `upgrade`, and `rollback`
- Multi-environment configuration files such as `values-dev.yaml` and `values-test.yaml`

In this article, we will bring these two technologies together to address the following key question:

**How does Argo CD integrate with Helm?**

In other words:

- Helm is responsible for templating and parameterization.
- Argo CD handles GitOps coordination.
- How do these two work together?

This document continues to build on your current setup:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- A Harbor private repository
- The `argocd` namespace with Argo CD installed
- A basic Helm Chart: `manual-web`
- Existing configuration files `values-dev.yaml` and `values-test.yaml`

## Tags

#Kubernetes #CI-CD #GitOps #ArgoCD #Helm #Chart #Values #Declarative Delivery #Practical Notes

## Learning Objectives

After completing this article, you should understand:

1. Which specific layer of Helm is actually integrated with Argo CD.
2. That Argo CD does not replace Helm but utilizes Helm’s capabilities within GitOps frameworks.
3. How `Chart + values` form the input for an Application in Argo CD.
4. How to apply your current `manual-web` Chart to understand the relationship between Argo CD and Helm.
5. The reasons why Helm and Argo CD work so well together.

## Main Experimental Approach

This article does not require you to set up a complete remote Git repository with Argo CD today and ensure successful synchronization. For now, focus on the following four tasks:

1. Clearly understand what happens when Argo CD integrates with Helm.
2. Use your existing Helm Chart directory to reverse-engineer the Application input structure.
3. Compare the logic of `helm template` with Argo CD’s approach to grasp their relationship.
4. Lay the foundation for eventually implementing Git repository-driven deployments.

---

## Part 1: Answering a Core Question – What Exactly Does Argo CD Integrate With Helm?

Many people initially misunderstand this as:

- Argo CD replacing Helm
- Or Helm replacing Argo CD

Neither is correct.

A more accurate understanding is:

### What Helm Does

- Organizes Charts
- Uses templates for structuring data
- Allows parameter customization with values files
- Generates the final Kubernetes YAML configuration.

### What Argo CD Does

- Retrieves the configuration from a Git repository
- Determines the desired target state
- Uses appropriate tools to generate the required state
- Compares this state with the current cluster configuration and ensures synchronization.

### Key Insight for This Step

The essence of integrating Argo CD with Helm is not:

“Replacing Helm”

but rather:

**In GitOps contexts, Argo CD utilizes Helm’s templating capabilities to generate the necessary Kubernetes configurations.**

---

## Part 2: Understanding Their Relationship in One Sentence

For now, remember this crucial point:

> **Helm handles the templating of configuration data, while Argo CD ensures that the resulting state is consistently synchronized with the cluster.**

This sentence encapsulates the core relationship between Helm and Argo CD.

---

## Part 3: Returning to Your Existing Helm Chart Directory

Enter the Helm Chart directory you have previously set up:

    cd ~/08-ci-cd/08-helm-lab/manual-web

Examine the current structure:

    ls
    ls templates
    cat Chart.yaml
    cat values.yaml
    cat values-dev.yaml
    cat values-test.yaml

You now have:

- `Chart.yaml`
- A `templates` directory
- `values.yaml` and `values-dev.yaml` files for different environments

### Key Insight for This Step

From Helm’s perspective, this entire set of files represents a:

- Installable and upgradable Chart.

From Argo CD’s viewpoint, these files will eventually serve as:

- The “target state input source” within a Git repository.

In other words, this configuration can be used to define an Application in Argo CD.

---

## Part 4: Identifying the Most Critical Input When Integrating Argo CD with Helm

Using your current understanding, the basic inputs for integrating Argo CD with Helm include:

### 1) Git Repository Address

This---

## Section Nine: The Four Dimensions Most Worth Clarifying at This Stage

In the future, whenever you think about integrating Argo CD with Helm, start by considering these four dimensions.

### 1) Repo

Specify which Git repository to use for configuration.

### 2) Path

Determine the path of the Chart within the repository.

### 3) Values

Identify which set of parameters applies to the current environment.

### 4) Destination

Specify which namespace or cluster to synchronize the changes to.

### Understanding Required at This Step

Once you have a clear understanding of these four dimensions, integrating Argo CD with Helm will no longer seem abstract.

---

## Section Ten: Why “Changing Values While Keeping the Chart Unchanged” Is Particularly Suitable for GitOps

You learned about this in Article 16:

- `values.yaml`
- `values-dev.yaml`
- `values-test.yaml`

Now, you’ll see how this structure fits perfectly with GitOps.

### Reason 1: Stable Templates

Charts are used to define the application structure and are typically not changed frequently.

### Reason 2: Clear Differentiation Between Environments

The main differences between environments involve:

- tag
- replicas
- namespace
- domain name/resource parameters

These factors make it suitable to include them in `values`.

### Reason 3: Clear Git Diff Results

If a version update only involves changing the `image.tag` in `values-test.yaml`, it becomes very clear when viewing the Git changes.

### Understanding Required at This Step

This is why Helm values are commonly used in GitOps:

**Because they make environmental changes more focused, easier to review, and easier to trace back.**

---

## Section Eleven: Things Not to Rush At This Stage

To maintain a proper pace, some tasks can be postponed for now.

### Tasks Not to Rush On

- Do not rush to connect to an actual remote Git repository or configure credentials.
- Do not rush to fully create an Application in the Argo CD UI.
- Do not rush to study all the details of Helm parameter passthrough.
- Do not rush to handle multi-cluster destinations.

### What Is Most Important at This Stage

1. First, understand how Argo CD integrates with Helm.
2. Map the existing Charts/values to future GitOps use cases.
3. Prepare for actually creating Applications later on.

### Understanding Required at This Step

At this point, it is more important to clarify these relationships than to immediately implement everything.

---

## Section Twelve: Conduct a “Manual Simulation of GitOps + Helm” Experiment

This step still does not rely on a remote Git repository but is very helpful.

### Step 1: Confirm the Current Values in the Test Environment

    cd ~/08-ci-cd/08-helm-lab/manual-web
    cat values-test.yaml

Assume the current value is:

    image:
      tag: "test-13"

### Step 2: Prepare a New Tag, Such as test-25

Return to the application directory, build, and push the new content:

    cd ~/08-ci-cd/01-manual-release

    cat > index.html <<'EOF'
    <html>
      <head>
        <meta charset="utf-8">
        <title>manual release test-25</title>
      </head>
      <body>
        <h1>Hello K8s CI/CD</h1>
        <p>version: test-25</p>
      </body>
    </html>
    EOF

    docker build -t manual-web:test-25 .
    docker tag manual-web:test-25 yourHarborDomain/test/manual-web:test-25
    docker push yourHarborDomain/test/manual-web:test-25

### Step 3: Modify Only values-test.yaml

Return to the Chart directory:

    cd ~/08-ci-cd/08-helm-lab/manual-web

Change `tag: "test-13"` to `tag: "test-25"`.

### Step 4: Check the Rendering Result

    helm template manual-web . -f values-test.yaml -n test

### Step 5: Execute the Upgrade

    helm upgrade manual-web-test . -f values-test.yaml -n test

### Step 6: Verify the Page Results

    kubectl -n test rollout status deployment/manual-web
    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

Enter and execute:

    wget -qO- http://manual-web.test.svc.cluster.local

### Understanding Required at This Step

What you just did is very similar to how GitOps operations will work in the future:

- You did not directly modify the Deployment or use `kubectl set image`.
- Instead, you changed the “target state configuration” (values) and let the system adjust accordingly.

In a GitOps scenario, the step of manually executing `