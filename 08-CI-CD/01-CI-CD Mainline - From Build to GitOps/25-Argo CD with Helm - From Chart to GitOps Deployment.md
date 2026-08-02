# 25-Argo CD Integration with Helm: From Chart to GitOps Deployment

## Documentation Notes

This article is the 25th note in the 08-CI-CD learning path.

The previous articles have gradually laid out the coreMain of Argo CD:

- Article 19: Why GitOps/Argo CD exists
- Article 22: Minimal Argo CD installation and entry observation
- Article 23: Three core objects - Application, Project, Repository

You've also previously learned about Helm:

- Chart, templates, values
- `helm install / upgrade / rollback`
- Multi-environment `values-dev.yaml / values-test.yaml`

This article begins to truly connect the two lines, focusing on solving the following question:

**How does Argo CD integrate with Helm?**

In other words:

- Helm handles templating and parameterization
- Argo CD handles GitOps alignment
- How do these two work together

This article continues to align with your currentMain and environment:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private registry
- `argocd` namespace has Argo CD installed
- Has a minimal Helm Chart: `manual-web`
- Has `values-dev.yaml / values-test.yaml`

## Tags

#Kubernetes #CI-CD #GitOps #ArgoCD #Helm #Chart #Values #DeclaredDelivery #I'llTakeYourNotes.

## Learning Objectives

After completing this article, you should understand:

1. Understand which layer Argo CD is actually connecting to when integrating with Helm
2. Understand that Argo CD is not replacing Helm, but calling Helm capabilities in GitOps mode
3. Understand how `Chart + values` becomes an Application input in Argo CD
4. Be able to use your current `manual-web` Chart to understand the relationship between Argo CD and Helm
5. Be able to explain why Helm and Argo CD easily cooperate

## This Article's ExperimentMain

This article doesn't require you to fully connect a remote Git repository to Argo CD and synchronize successfully today.  
At this stage, the focus is on 4 things:

1. First explain clearly what Argo CD does when integrating with Helm
2. Use your existing Helm Chart directory to reverse-engineer Application inputs
3. Through `helm template` and Argo CD thinking, understand their relationship
4. Lay the foundation for future Git repository-driven deployments

---

## Part 1: First answer a core question - What does Argo CD connect to when integrating with Helm

Many people initially misunderstand this as:

- Argo CD replacing Helm
- Or Helm replacing Argo CD

This is inaccurate.

A more accurate understanding is:

### What Helm handles

- Chart organization
- templates templating
- values parameter override
- Generate final K8s YAML

### What Argo CD handles

- Watch the configuration source in Git
- Determine what the target state is
- Call the appropriate tools to render the target state
- Compare with the cluster's current state and synchronize

### Understanding to establish at this stage

So the essence of Argo CD integrating with Helm is not:

"Discard Helm"

But rather:

**In GitOps mode, Argo CD uses Helm as a configuration rendering capability.**

---

## Part 2: Understand the relationship between the two with one sentence

At this stage, remember this most important sentence:

> **Helm solves "how to template configurations", Argo CD solves "how to continuously align the templated target state to the cluster".**

This sentence is the main thread of this article.

---

## Part 3: Return to your existing Helm Chart directory

Enter the Helm Chart directory you've already worked on:

    cd ~/08-ci-cd/08-helm-lab/manual-web

Check the current structure:

    ls
    ls templates
    cat Chart.yaml
    cat values.yaml
    cat values-dev.yaml
    cat values-test.yaml

You now have:

- `Chart.yaml`
- `templates/`
- `values.yaml`
- `values-dev.yaml`
- `values-test.yaml`

### Understanding to establish at this stage

From a Helm perspective, this entire directory is:

- A installable, upgradable Chart

From an Argo CD perspective, it will eventually be:

- A "target state input source" in a Git repository

In other words, this entire setup can later become the configuration source for an Application.

---

## Part 4: What is the most core input when Argo CD integrates with Helm

Using your currentMain to understand, a minimal Argo CD + Helm input typically includes:

### 1) Git repository address

Tell Argo CD:

- Where to get the configuration

### 2) Path in the repository (path)

Tell Argo CD:

- Where the Helm Chart is located in the repository

In your current experiment, you can first imagine:

- `08-helm-lab/manual-web`

### 3) Target environment corresponding values

Tell Argo CD:

- Which values file to use for dev
- Which values file to use for test

For example:

- `values-dev.yaml`
- `values-test.yaml`

### Understanding to establish at this stage

So when Argo CD integrates with Helm, its most common working method is actually:

- Find the Chart in the Git repository
- Decide which values file to use
- Render the final YAML
- Then synchronize with the cluster

---

## Part 5: Use `helm template` to reverse-engineer what Argo CD does

This section is important because it makes Argo CD's behavior visible.

### Step 1: First render the test environment

Run the following in the Chart root directory:

    helm template manual-web . -f values-test.yaml -n test

### Step 2: Then render the dev environment

    helm template manual-web . -f values-dev.yaml -n dev

### What to focus on at this stage

You should focus on observing:

- Deployment's image
- namespace
- replicas
- Service's namespace / port

### Understanding to establish at this stage

You'll find that:

- Same Chart
- Different values
- Different rendering results

This is actually one of the key underlying logic when Argo CD integrates with Helm:

**It doesn't directly send the Chart as-is into the cluster, but first renders the target state for the current environment by combining it with values.**

---

## Part 6: Differences Between Helm Self-Deployment and Argo CD Integrating with Helm

This question is particularly important.

### Method 1: Manually Using Helm to Deploy

For example, you've done something like:

    helm install manual-web-test . -f values-test.yaml -n test

Or:

    helm upgrade manual-web-test . -f values-test.yaml -n test

The characteristics of this mode are:

- You actively execute helm commands
- Helm renders and updates the cluster immediately
- The deployment action is initiated by you

### Method 2: Argo CD Integrating with Helm

In this mode, it's typically:

- The Chart and values are stored in a Git repository
- Argo CD watches the Git repository
- Argo CD is responsible for invoking Helm rendering logic
- Argo CD then aligns the cluster to the rendered result

The characteristics of this mode are:

- You don't manually execute helm commands each time
- Instead, Git state changes drive Application state changes

### Understanding to Establish at This Stage

So the boundary between Helm and Argo CD can be viewed as:

- Pure Helm: More like you actively issuing a "templated deployment command"
- Argo CD + Helm: More like the controller takes over and continuously synchronizes after Chart/values changes in Git

---

## Part 7: Why Helm and Argo CD Particularly Work Well Together

Because they naturally solve problems in aUp and down. (upstream-downstream) relationship.

### What Helm Excels At

- Consolidating repetitive YAML into templates
- Expressing environment differences with values
- Making configurations more maintainable

### What Argo CD Excels At

- Continuously aligning declared configuration states in Git to the cluster
- Detecting drift
- Managing synchronization status and health status

### Understanding to Establish at This Stage

This is like:

- Helm first organizes the "target state content"
- Argo CD then ensures the "target state is implemented and continuously maintained"

Thus, the combination of the two is very natural.

---

## Part 8: Create a Minimal Environment Mapping with Current `manual-web`

This section brings the concept back to your current directory.

Assume you eventually place your current Helm directory into a Git repository.

### For the dev Environment

The Application input Argo CD sees can be understood as:

- Repository: A certain Git repository
- Path: `08-helm-lab/manual-web`
- Values: `values-dev.yaml`
- Destination Namespace: `dev`

### For the test Environment

The Application input Argo CD sees can be understood as:

- Repository: The same Git repository
- Path: `08-helm-lab/manual-web`
- Values: `values-test.yaml`
- Destination Namespace: `test`

### Understanding to Establish at This Stage

In other words:

- The same Chart
- The same Git repository
- Can be treated as input sources for multiple environments by Argo CD
- The key difference is values and destination

---

## Part 9: The 4 Dimensions Most Worth Clarifying at This Stage

In the future, whenever you think about Argo CD integrating with Helm, start by examining these 4 dimensions first.

### 1) Repo

Where the configuration comes from in terms of Git repository.

### 2) Path

Where the Chart is located in the repository.

### 3) Values

Which set of parameters to override for the current environment.

### 4) Destination

Where to synchronize to (namespace/cluster).

### Understanding to Establish at This Stage

Once these 4 dimensions are clear, the act of Argo CD integrating with Helm will no longer seem abstract.

---

## Part 10: Why "Chart Unchanged, values Changed" Particularly Suits GitOps

You've already learned in the 16th article:

- `values.yaml`
- `values-dev.yaml`
- `values-test.yaml`

Now looking back, you'll find this structure aligns very well with GitOps.

### Reason 1: Stable Templates

Charts are responsible for expressing application structure, and their change frequency is typically not very high.

### Reason 2: Clear Environment Differences

Different environments mainly modify:

- tag
- replicas
- namespace
- domain/resource parameters

These are all well-suited for putting into values.

### Reason 3: Clearer Git diffs

If a version release only changes:

- `values-test.yaml`'s `image.tag`

Then the changes in Git will be very clear.

### Understanding to Establish at This Stage

This is why Helm values are common in GitOps:

**Because it focuses environment changes, makes them easier to review, and easier to trace back.**

---

## Part 11: Things Not to Rush Into at This Stage

To maintain the pace, some things will be postponed in this article.

### Things Not to Rush Into Now

- Don't rush to connect to a real remote Git repository and configure credentials
- Don't rush to fully create an Application in the Argo CD UI
- Don't rush to study all details of Helm parameter passing
- Don't rush to handle multi-cluster destinations

### What's Most Important Now

1. First understand how Argo CD integrates with Helm
2. First align your existing Chart/values with future GitOps entry points
3. Prepare for creating the Application later

### Understanding to Establish at This Stage

Now, just get the relationships clear, which is more important than immediately connecting everything.

---

## Part 12: Do a Minimal Experiment of "Manual Simulation of GitOps + Helm"

This section still doesn't rely on a remote Git, but it's very helpful.

### Step 1: Confirm Current values for the test environment

    cd ~/08-ci-cd/08-helm-lab/manual-web
    cat values-test.yaml

Assume the current content is:

    image:
      tag: "test-13"

### Step 2: Prepare a new tag, e.g., test-25

First return to the application directory, build and push new content:

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
docker tag manual-web:test-25 your-Harbor-domain/test/manual-web:test-25
docker push your-Harbor-domain/test/manual-web:test-25

### Step 3: Only modify values-test.yaml

Return to the Chart directory:

    cd ~/08-ci-cd/08-helm-lab/manual-web

Change:

    tag: "test-13"

To:

    tag: "test-25"

### Step 4: First view the rendered result

    helm template manual-web . -f values-test.yaml -n test

### Step 5: Then execute upgrade

    helm upgrade manual-web-test . -f values-test.yaml -n test

### Step 6: Verify the page result

    kubectl -n test rollout status deployment/manual-web
    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

After entering, execute:

    wget -qO- http://manual-web.test.svc.cluster.local

### Current understanding to establish

What you just did is already very close to the future GitOps operational habits:

- Do not manually edit Deployments directly
- Do not directly `kubectl set image`
- Instead modify "target state configuration" (values)
- Then let the system align

In GitOps scenarios, the step of "manually executing helm upgrade" is further delegated to Argo CD.

---

## Part 13: Now summarize backwards, the essence of Argo CD integrating with Helm

At this point, you should be able to naturally explain:

### Not

- Argo CD replaces Helm
- Or Helm replaces Argo CD

### But

- Helm is responsible for rendering Chart + values into the final target state
- Argo CD is responsible for continuously reading this input from Git and aligning it with the cluster

### Current understanding to establish

Therefore, the essence of Argo CD integrating with Helm is:

**Argo CD uses Helm as a target state rendering capability.**

---

## Part 14: This section's practice

### Practice 1: Perform an Argo CD input mapping using the current `manual-web` directory

Clearly write:

- What is the Repository
- What is the Path
- What environment does values-dev.yaml correspond to
- What environment does values-test.yaml correspond to
- What are the destination namespace for each

### Practice 2: Perform another release by only modifying values

Requirements:

- Prepare a new tag
- Only modify `values-test.yaml`
- First `helm template`
- Then `helm upgrade`
- Then verify the page content

### Practice 3: Answer the following 5 questions yourself

1. When Argo CD integrates with Helm, which layer is it actually integrating with
2. What is the boundary between Helm and Argo CD
3. Why are Helm values particularly suitable for GitOps
4. Why can the same Chart correspond to multiple Application environments
5. Why is "modifying values" more suitable for subsequent GitOps than "directly manually editing Deployment"

If you can explain these 5 questions yourself, you've mastered this section.

---

## Content to be able to explain after this section

After completing this section, it's recommended to be able to clearly explain the following:

When Argo CD integrates with Helm, it's not replacing Helm, but utilizing Helm's template rendering capability in GitOps mode.  
Helm is responsible for combining Chart and values into the final K8s target state, while Argo CD is responsible for reading this input from a Git repository and continuously aligning the cluster state to the rendered result.  
For the same application, multiple environment Application inputs can be formed by using the same Chart with different values files, such as dev, test, etc.  
Therefore, the relationship between Helm and Argo CD is not a competitive relationship, but aUp and down. relationship between template rendering capability and target state control capability.

## Common Issues and Troubleshooting Directions

### Issue 1: I already know Helm, why do I need to understand Argo CD integrating with Helm

Because Helm solves template rendering,  
Argo CD further solves:

- Git target state management
- Continuous synchronization
- Drift governance

### Issue 2: Will we no longer need helm commands in the future

No.  
helm commands are still valuable tools for understanding templates, rendering results, and manual verification.  
In formal GitOps scenarios, the continuous synchronization action is more likely to be taken over by Argo CD.

### Issue 3: Why this section didn't directly create a complete Argo CD Application

Because the most important thing at this stage is to first clarify the relationship between Helm input and Argo CD,  
In the next step, creating a Git repository-driven release will be smoother.

---

## Key Points of This Section

1. Core relationship of Argo CD integrating with Helm
2. Role of Chart / values in GitOps
3. Four key dimensions: Repository + Path + Values + Destination
4. Why values are particularly suitable for multi-environment GitOps
5. Why Helm and Argo CD are easy to cooperate

## One-Sentence Summary

The essence of Argo CD integrating with Helm is not replacing Helm, but transforming Helm into a rendering capability for GitOps target states, and then letting Argo CD continuously align the rendered results to the Kubernetes cluster.