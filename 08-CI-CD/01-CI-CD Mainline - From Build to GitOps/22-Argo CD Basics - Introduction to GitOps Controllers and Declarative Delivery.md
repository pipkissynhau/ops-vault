# 22-Argo CD Basics: Introduction to GitOps Controllers and Declarative Delivery

## Documentation Note

This article is the 22nd note in the 08-CI-CD learning pathway.

In the previous 19th article, we established the basic understanding of GitOps, which includes:

- Git can serve as the source for the target state.
- The actual state of the cluster may deviate from this target state.
- A controller is needed to continuously align the Git state with the cluster state.

In this article, we will implement this “controller” using a specific tool:

**Argo CD.**

This article does not require you to set up Argo CD in a production environment today, nor do you need to master all its object models immediately.  
The focus at this stage is:

1. To understand the role of Argo CD within the overall deployment process.
2. To clarify its boundaries compared to previously learned tools such as kubectl, Helm, GitLab CI, and Jenkins.
3. To perform a minimal installation and observation experiment.
4. To first understand what the “GitOps controller” actually does.

This article continues to use the current main environment and experimental setup:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private repository
- `test` namespace
- The minimum application `manual-web` has already been built, pushed, deployed using kubectl/Helm, rolled back, and tested in multiple environments.

## Tags

#Kubernetes #CI-CD #GitOps #ArgoCD #Helm #Declarative Delivery #Configuration Alignment #Controller #Practical Notes

## Learning Objectives

After completing this article, you should be able to:

1. Understand the position of Argo CD within the entire deployment pipeline.
2. Clarify the differences between Argo CD and previously learned tools like GitLab CI/Jenkins/Helm.
3. Recognize the core responsibilities of Argo CD.
4. Perform a minimal installation in your current cluster.
5. Access the Argo CD interface and observe its basic status.
6. Explain why Argo CD is considered a GitOps controller, rather than just a replacement for kubectl for deployment tasks.

## Experimental Approach for This Article

This article is divided into 4 sections:

1. First, place Argo CD within the entire deployment process you have already learned about.
2. Perform a minimal installation in your current cluster.
3. Observe the core resources created after installing Argo CD.
4. Use the existing Helm/values directories to get a sense of how “who will take over managing the target state” in the future.

---

## Section 1: Placing Argo CD within the Entire Deployment Process

You have already learned about the following tools:

- GitLab CI/Jenkins: These focus more on the automation of the early stages of the deployment process.
- Harbor: Used for storing images.
- Helm: Used for templating and parameterizing configurations.
- kubectl/Helm upgrade: Used to actively push configuration changes to the cluster.
- GitOps: Uses Git as the source for the target state.

Now, the most logical place to integrate Argo CD into this process is:

Code  
→ CI build to generate images  
→ Push images to Harbor  
→ Update the deployment configuration in Git  
→ Argo CD reads the Git configuration  
→ Argo CD ensures the cluster state aligns with the Git target state

### Key Concept to Understand at This Step

Argo CD does not replace the following steps:

- Docker build
- Harbor image push
- Image tag management

Its main role is to address the question of “who will continuously ensure that the cluster state aligns with the prepared images and configurations after they are ready.”

---

## Section 2: Understanding the Boundaries Between Argo CD and Previously Learned Tools

It is essential to clearly define these boundaries to avoid confusion later on.

### What do GitLab CI/Jenkins Do?

They mainly handle:

- Code retrieval
- Testing
- Building
- Tag creation
- Image pushing to Harbor
- Sometimes, they also trigger configuration updates.

### What does Helm Do?

Helm is primarily used for:

- Templating and parameterizing configurations.
- Organizing configurations in Charts.
- Overriding values using the `values` directory.

### What does kubectl/Helm upgrade Do?

This tool is mainly used to:

- Manually execute a deployment command.

### What does Argo CD Do?

Argo CD focuses on:

- Continuously monitoring whether the Git configuration matches the cluster state.
- Performing synchronization when there are discrepancies.
- Ensuring that the target state remains consistent over time.

### Key Concept to Understand at This Step

The difference between Argo CD and other tools is that it does not simply “execute a command once.” Instead, it:

- Continuously monitors the status.
- Continuously compares the two states.
- Continuously3. Entries or status terms related to synchronization (sync)
4. Entries or status terms related to health status

### Understanding required for this step

You don't need to study all the pages in detail right away. First, understand that:

- Argo CD organizes its interface around "application status" and "synchronization status."

This aligns with its role as a GitOps controller.

---

## Part Twelve: Treating your existing Helm directory as the target state directory that will be managed by Argo CD in the future

This part is very important because it connects the previous steps with Argo CD.

Enter your existing Helm directory:

    cd ~/08-ci-cd/08-helm-lab/manual-web

You now have:

- `Chart.yaml`
- `templates/`
- `values-dev.yaml`
- `values-test.yaml`

### How to understand this step

In previous chapters 8, 13, and 16, you learned how to:

- Use Helm Charts to represent application templates
- Use values files to reflect environmental differences
- Use `helm upgrade` to actively push the state to the cluster

In the GitOps/Argo CD context, this set of directories can be further developed into:

**The target state source in Git.**

In other words, you won't have to manually execute `helm upgrade ...` in the future. Instead, you will:

- Modify the values or Charts in Git
- Let Argo CD detect these changes
- Then let Argo CD perform the synchronization

### Understanding required for this step

The Helm knowledge you learned earlier is not wasted;  
in fact, it is a common configuration format used by Argo CD.

---

## Part Thirteen: How to understand "declarative delivery" at this stage

The title of this chapter includes the term "declarative delivery." You need to place this concept in a position you can grasp first.

### Consider how you have done it before

For example:

    kubectl -n test set image deployment/manual-web manual-web=yourHarbor域名/test/manual-web:v10

This is more like:

- I actively issue a command to change the cluster state.

### Now, consider the GitOps/Argo CD approach

It's more like:

- I declare what the environment should be in Git
- The controller is responsible for making it that way and maintaining it as much as possible.

### Understanding required for this step

Declarative delivery isn't just about using fewer commands;  
it's about:

**Focusing on the final desired state, rather than manually specifying every step to get there.**

---

## Part Fourteen: What not to do at this stage

This part is important to avoid getting overwhelmed by complexity too early.

### Things you don't need to rush into:

- Don't rush to integrate the entire Git repository immediately.
- Don't try to set up a complete GitOps environment for dev/test/prod today.
- Don't rush to study all Argo CD components.
- Don't try to configure all synchronization strategies right away.

### The most important things at this stage are:

1. Install it.
2. Make sure you can see it and understand how it works in the cluster.
3. Understand which part of the main workflow it will be responsible for managing in the future.

---

## Part Fifteen: Looking back now, why do we still need Argo CD?

By this point, you should be able to answer this question.

### It's not because the previous tools are useless

- GitLab CI/Jenkins are still valuable.
- Harbor is still useful.
- Helm is still useful.
- kubectl is still useful.

### It's because there is still a missing component

What's missing is:

- A system that continuously monitors the differences between Git and cluster states.
- A system that detects any drifts.
- A system that performs synchronization.
- A system that ensures the target state is always maintained.

This is where Argo CD comes in.

### Understanding required for this step

Argo CD doesn't duplicate what has already been done;  
instead, it adds:

**The layer of "continuous alignment with the target state" to the entire workflow.**

---

## Part Sixteen: This chapter's exercises

### Exercise 1: Independently complete the installation and access of Argo CD

Requirements:

- Create a `argocd` namespace.
- Use the official installation instructions.
- Ensure that the main Pods are ready.
- Open the interface through port-forwarding.
- Obtain the initial password and log in.

### Exercise 2: Observe which core components Argo CD creates in the cluster

Requirements:

- Check `deploy`.
- Check `svc`.
- Check `pods`.
- Identify the most critical components.

### Exercise 3: Answer the following 5 questions on your own

1. What is Argo CD's role in the entire process?
2. What is the boundary between Argo CD and GitLab CI