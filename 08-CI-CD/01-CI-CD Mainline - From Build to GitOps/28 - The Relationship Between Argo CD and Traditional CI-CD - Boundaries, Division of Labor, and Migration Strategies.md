# 28 - The Relationship Between Argo CD and Traditional CI-CD: Boundaries, Division of Labor, and Migration Strategies

## Document Description

This article is the 28th note in the 08-CI-CD learning pathway and also the final one for this phase.

In the previous articles from 01-27, we have covered the entire main line from basics all the way to GitOps, including:

- Manual deployment processes
- Image building and tagging
- Harbor repository management
- GitLab CI / Jenkins Pipelines
- Deployment via Helm and rollback mechanisms
- Multi-environment publishing
- Security fundamentals
- Advanced Harbor usage
- GitOps basics
- Installation of Argo CD, core components, integration with Helm, and use of Git repositories for deployment and troubleshooting

In this article, we will not introduce any new tools. Instead, our focus will be on one thing:

**To place Argo CD and traditional CI/CD within the same framework and clarify their relationship.**

The goal of this article is not to redefine these concepts but to help you connect what you have learned so far and address the following practical questions:

- If we already have GitLab CI / Jenkins, why do we still need Argo CD?
- Does having Argo CD mean we no longer need GitLab CI / Jenkins?
- What exactly do CI, Helm, Harbor, and Argo CD do?
- When is it appropriate to continue using traditional push-based deployment methods?
- When is it time to gradually transition to GitOps?
- If both systems coexist in your work environment at present, how should you understand their roles?

This article will continue to align with your current main line and experimental setup:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Private Harbor repository
- The `argocd` namespace has already installed Argo CD
- You have completed experiments with manual deployment, Helm deployment, and GitOps deployment for the minimal application `manual-web`.

## Tags

#Kubernetes #CI-CD #GitOps #ArgoCD #Jenkins #GitLabCI #Helm #Harbor #Deployment Systems #Practical Notes

## Learning Objectives

After completing this article, you should be able to:

1. Understand the boundaries between traditional CI/CD and Argo CD.
2. Recognize how “CI” and “CD” are redefined in the GitOps era.
3. Identify the roles of Helm, Harbor, GitLab CI, Jenkins, and Argo CD within the entire deployment process.
4. Comprehend why many teams do not switch entirely but instead use both systems simultaneously for a period of time.
5. Explain the minimum migration steps when transitioning from traditional deployment to GitOps.
6. Present the entire main line learned so far in a coherent manner.

## Main Content of this Article

This article is divided into 4 sections:

1. First, place traditional CI/CD and GitOps within the same framework.
2. Then, discuss their boundaries and division of labor.
3. Next, examine the most common migration methods used in real teams.
4. Finally, conduct a comprehensive review of the entire main line.

---

## Section 1: Replacing What You Have Learned Within the Same Framework

Let’s start by considering what you have actually done so far.

You have personally performed these actions:

- Editing `index.html`
- Running `docker build`
- Applying tags to images with `docker tag`
- Pushing images to Harbor using `docker push`
- Verifying images on the Harbor page
- Using `kubectl set image` to update images
- Upgrading Helm applications
- Modifying `values-test.yaml` files through Git repositories
- Using Argo CD to synchronize applications

If we place these actions within a complete deployment framework, they can be organized as follows:

Application content  
→ Image building  
→ Tagging  
→ Pushing to Harbor  
→ Updating deployment configurations (YAML / Helm values / Git)  
→ Cluster deployment  
→ Status verification  
→ Drift management and continuous alignment

### Understanding Required at This Step

By now, you are no longer just proficient in a single tool but have developed the ability to view the entire deployment process as a cohesive whole.

The focus of this section is to clearly define the roles of “traditional CI/CD” and “Argo CD” within this framework.

---

## Section 2: Identifying Where Traditional CI/CD Excels

When you learned about GitLab CI and Jenkins Pipelines, you were already familiar with these tasks:

- Pulling code from repositories
- Building applications
- Running `docker build`
- Applying tags to images with `docker tag`
- Pushing images to Harbor using `docker push`
- Sometimes directly executing `kubectl` or `helm` commands for deployment

At this stage, traditional CI/CD excels in the first half of the deployment process,## Section 9: What is the Most Suitable "Boundary Scope" for Your Current Mainline?

It is recommended that you fix the following scope now, as it will be very useful in your future work.

### GitLab CI / Jenkins

More focused on:

- Building and producing artifacts

### Harbor

More focused on:

- Image storage and governance

### Helm

More focused on:

- Configuring templates and parameters

### Argo CD

More focused on:

- Maintaining continuous alignment with Git target states

### kubectl

More focused on:

- Observation, troubleshooting, and basic operations

### Understanding to Establish at This Step

As long as you clearly define this boundary,
you won't encounter confusion when discussing solutions in interviews or during work later on.9. Understand CI/CD security and advanced governance using Harbor.
10. Comprehend release strategies such as blue-green deployment, canary testing, and grayboxing.
11. Grasp the step-by-step approach to troubleshooting common CI/CD issues.
12. Recognize the goal-state approach of GitOps.
13. Learn about the installation process of Argo CD, its object management, integration with Helm, and how it uses Git repositories for deployment and troubleshooting.
14. Ultimately understand the differences and roles between Argo CD and traditional CI/CD systems.

### Understanding Required at This Phase

This represents a complete mainline from image creation to cluster deployment, ranging from manual releases to GitOps-based processes.

---

## II. If You Were to Explain “Minimum Delivery Pipeline” Now, It Would Be Recommended to Do So This Way

You could try rephrasing it like this:

In the Kubernetes context, application delivery typically begins with changes in code or content. These changes are first compiled into an image using a Dockerfile, then labeled with appropriate tags for version tracking, and subsequently pushed to Harbor as a centralized image repository.  
The configuration layer is defined using YAML or Helm Charts along with their corresponding values files to specify the desired target states for different environments.  
For traditional approaches, tools like GitLab CI or Jenkins can execute `kubectl` or `helm` commands directly after completing the build and push process to deploy changes to the cluster. In contrast, GitOps utilizes Argo CD to continuously monitor the configuration targets in Git and synchronize them with Kubernetes.  
After deployment, it is necessary to monitor processes such as rollout, Pod status, Service functionality, and business outcomes, and have the capability to perform rollbacks and troubleshooting if needed.

If you can explain this smoothly, it indicates that you have effectively established a systematic approach at this stage.

---

## III. The Actions Most Worth Practicing Repeatedly at This Phase

Not all knowledge points require equal repetition; here are the key ones that deserve special attention:

### 1) Build / Tag / Push

Specifically:

- `docker build`
- `docker tag`
- `docker push`

### 2) Deployment Updates and Rollout Monitoring

Specifically:

- `kubectl set image`
- `kubectl rollout status`
- `kubectl describe pod`

### 3) Helm Values Modifications and Upgrades

Specifically:

- `helm template`
- `helm upgrade`
- `helm history`
- `helm rollback`

### 4) The Minimum GitOps Change Chain with Argo CD

Specifically:

- Modify `values-test.yaml`
- Commit and push the changes
- Check for OutOfSync in Argo CD
- Synchronize the changes
- Verify business functionality

### 5) Troubleshooting Sequence

This involves breaking down issues into specific stages:

- Git / Configuration
- Build
- Push
- Sync / Deployment
- Pod / Rollout
- Business Results

---

## IV. What Is the Biggest Change You Have Experienced Since This Phase Began?

Initially, you might have only known a few individual commands and been familiar with terms like Jenkins, GitLab CI, Harbor, Helm, and Argo CD, but not necessarily how they worked together. Now, you should be able to:

### 1) Understand How the Entire Process Works

Instead of viewing these tools in isolation, you should recognize how they are interconnected—from code to images, from repositories to clusters, and from controllers to business outcomes.

### 2) Know What Each Tool Does

You will no longer easily confuse concepts like CI, Harbor, Helm, and Argo CD.

### 3) Identify Where Problems Occur

This is a significant advancement. It’s more valuable than simply memorizing more commands.

### 4) Adopt a “Platform-Oriented” Perspective

You are no longer just focused on deploying applications; you are beginning to understand how the entire delivery system is designed and implemented.

---

## V. What Is the Natural Next Step After This Phase?

If you wish to advance further, the natural next steps might include:

### Direction 1: Turning the Current Mainline into a Real-World Project

For example:

- Implementing a full-endchain involving frontend, API, Nginx, ConfigMap, and Ingress.
- Integrating GitLab CI or Jenkins effectively.
- Enabling automatic synchronization with Argo CD.

### Direction 2: Enhancing K8s Deployment-related Capabilities

For example:

- Optimizing Ingress management, domain name configuration, and TLS settings.
- Understanding how ConfigMap and Secret management affect deployments.
- Investigating the impact of HPA, PDB, Readiness, and Liveness metrics on deployment processes.
- Ensuring resource constraints do not compromise deployment stability.

### Direction 3: Deepening GitOps Practices

For example:

- Exploring more details about Application, Project, and Repository concepts.
- Implementing advanced features like Sync, Diff, Health checks, and self-healing