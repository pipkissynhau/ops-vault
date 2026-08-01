# 28-Relationship Between Argo CD and Traditional CI/CD: Boundaries, Division of Labor, and Migration Strategies

## Document Notes

This article is the 28th note in the 08-CI-CD learning path, and also the final article in this phase.

The previous 01-27 articles have taken the main line from basics all the way to GitOps, including:

- Manual release pipeline
- Image building and tagging
- Harbor repository
- GitLab CI / Jenkins Pipeline
- Deployment / Helm release and rollback
- Multi-environment release
- Security basics
- Harbor advanced features
- GitOps basics
- Argo CD installation, core objects, integration with Helm, Git repository-driven release and troubleshooting

In this article, no new tools are introduced. The focus is only on one thing:

**Putting Argo CD and traditional CI/CD on the same diagram to clearly explain their relationship.**

The goal of this article is not to re-define concepts, but to help you connect what you've learned before, answering these practical questions:

- Why do we still need Argo CD if we have GitLab CI / Jenkins?
- Does having Argo CD mean we no longer need GitLab CI / Jenkins?
- What are the specific roles of CI, Helm, Harbor, GitLab CI, Jenkins, and Argo CD in the entire pipeline?
- When is it suitable to continue using traditional push-type releases?
- When is it suitable to gradually migrate to GitOps?
- If you encounter both systems coexisting in the future, how should you understand them?

This article continues to align with your current main line and experimental environment:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private repository
- `argocd` namespace has Argo CD installed
- The minimum application `manual-web` has been manually released, Helm released, and GitOps released

## Tags

#Kubernetes #CI-CD #GitOps #ArgoCD #Jenkins #GitLabCI #Helm #Harbor #DisseminationSystem #I'llTakeYourNotes.

## Learning Objectives

After completing this article, you should be able to:

1. Understand the boundaries between traditional CI/CD and Argo CD
2. Understand how "CI" and "CD" are re-divided in the GitOps era
3. Understand the specific roles of Helm, Harbor, GitLab CI, Jenkins, and Argo CD in the entire pipeline
4. Understand why many teams do not "switch all at once" but instead "coexist with two systems for a period"
5. Be able to explain the minimal migration strategy for gradually transitioning from traditional releases to GitOps
6. Be able to explain the entire main line from 01-28

## Main Line of This Article

This article is divided into 4 sections:

1. First, place traditional CI/CD and GitOps on the same pipeline diagram
2. Then explain their boundaries and division of labor
3. Then explain the most common migration methods in actual teams
4. Finally, do a comprehensive review of the entire main line

---

## Part 1: First, Place All Previously Learned Content on the Same Pipeline

Start from the actions you've already done hands-on.

You've manually done these actions:

- Modify `index.html`
- `docker build`
- `docker tag`
- `docker push`
- Confirm the image on Harbor's page
- `kubectl set image`
- `helm upgrade`
- Modify `values-test.yaml` through the Git repository
- Let Argo CD synchronize the Application

If these actions are placed back into a complete pipeline, they can be organized as follows:

Application content  
→ Build image  
→ Tag  
→ Push Harbor  
→ Modify deployment configuration (YAML / Helm values / Git)  
→ Cluster release  
→ Status verification  
→ Drift governance and continuous alignment

### Understanding to Establish at This Stage

By now, you are no longer just skilled in a single tool,  
but you already have the ability to view problems from the entire pipeline.

The focus of this article is to place "traditional CI/CD" and "Argo CD" in their correct positions in this pipeline.

---

## Part 2: First, See What Traditional CI/CD Is Better At

When you studied GitLab CI and Jenkins Pipeline earlier, you've repeatedly encountered these actions:

- Pull code
- Build
- docker build
- docker tag
- docker push
- Sometimes directly execute kubectl / helm release

From the current stage, traditional CI/CD is best at the first half of automation.

### More Typical Responsibilities

#### 1) Pull Code

Get source code from the Git repository.

#### 2) Testing and Building

For example:

- Unit testing
- Packaging
- Build image

#### 3) Image Repository

For example:

- Tag
- Push Harbor

#### 4) Sometimes also responsible for "active release"

For example, directly execute:

- `kubectl apply`
- `kubectl set image`
- `helm upgrade`

### Understanding to Establish at This Stage

The strengths of traditional CI/CD are very clear:

**Turn "code" into "deployable artifacts" and be able to send them out seamlessly.**

---

## Part 3: Now See What Argo CD Is Better At

From the previous articles 19, 22, 25, 26, and 27, you've gradually established Argo CD's positioning.

Argo CD is better at:

- Not docker build
- Not push Harbor
- Not image testing

But rather:

- Read the target state from Git
- Compare Git and cluster states
- Do synchronization
- Handle OutOfSync
- Manage drift
- Ensure the target state remains valid

### More Typical Responsibilities

#### 1) Monitor Target State

What configuration is written in the Git repository.

#### 2) Monitor Current Cluster State

What is actually running in K8s.

#### 3) Do State Comparison

- Synced
- OutOfSync
- SyncFailed

#### 4) Do Synchronization and Return to Target State

- Manual Sync
- Can also be automatic Sync / Self-Heal later

### Understanding to Establish at This Stage

Argo CD is more focused on the latter half, especially good at:

**Making "what it should be" persist in the cluster long-term.**

---

## Part 4: Why Argo CD Is Not Here to Replace CI

This point must be clearly explained.

Many people who first encounter GitOps tend to misunderstand:

- Having Argo CD means Jenkins / GitLab CI is no longer useful

This understanding is incomplete.

### Because Argo CD Does Not Handle These Tasks

It usually does not handle:

- Code compilation
- Unit testing
- Docker build
- Push Harbor
- Image tag design
- Pre-steps for security scanning

These tasks still suit better:

- GitLab CI
- Jenkins
- Other CI systems

### Understanding to Establish at This Stage

Argo CD is not a "full-stack replacement platform,"  
It mainly complements:

**The continuous alignment of configuration states.**

---

## Part 5: Why Argo CD Is Not Simply Replacing kubectl / Helm

Previously, you've done: /think

- `kubectl set image`
- `helm upgrade`
- `helm template`

These capabilities remain effective even in the GitOps era.

### The Value of kubectl Remains

It remains:

- A fundamental troubleshooting tool
- A resource observation tool
- An emergency localization tool

### The Value of Helm Remains

It remains:

- A templating tool
- A parameterization tool
- A Chart organization tool
- One of the key input formats for Argo CD

### Current Understanding to Establish

Argo CD is not a replacement for kubectl/Helm,  
but rather elevates:

- kubectl's "manual direct cluster modification"
- Helm's "manual active upgrade"

to:

**Define target state in Git, with controllers continuously aligning.**

---

## Part 6: Viewing Both Approaches Side by Side

This section is the most critical comparison in the entire text.

### Traditional Push-Based Release Approach

Code change  
→ CI build  
→ push Harbor  
→ pipeline executes kubectl/helm directly  
→ cluster state changes

Characteristics:

- Active push
- Release entry point in pipeline/manual command
- Cluster doesn't continuously monitor Git state

### GitOps/Argo CD Approach

Code change  
→ CI build  
→ push Harbor  
→ modify deployment configuration in Git  
→ Argo CD detects Git-cluster inconsistency  
→ Argo CD synchronizes  
→ cluster state aligns with Git

Characteristics:

- Git becomes the target state source
- Release entry point in Git configuration change
- Controller continuously monitors state

### Current Understanding to Establish

The biggest difference between the two approaches isn't "whether there's a pipeline,"  
but rather:

**Whether the final release entry point is a command or Git target state.**

---

## Part 7: Why Many Teams Coexist Both Approaches

This point closely reflects real-world work environments.

Many teams don't switch overnight from:

- Jenkins + kubectl

to:

- GitOps + Argo CD

A more common reality is:

### Stage 1: Traditional Release Dominance

- Jenkins/GitLab CI build
- Direct `kubectl apply`
- Direct `helm upgrade`

### Stage 2: Partial GitOps Adoption

- dev/test use Argo CD first
- prod still uses traditional methods conservatively
- Or vice versa, only stable projects are adopted first

### Stage 3: Gradual Shift to GitOps

- CI still handles build/push
- CD increasingly delegated to Argo CD

### Current Understanding to Establish

The most common scenario isn't "pure A or pure B,"  
but rather:

**Traditional release and GitOps coexist in the same company for a long time.**

---

## Part 8: Understanding CI/CD Redefinition in GitOps Era

This is particularly important.

### In Traditional Terminology

CI/CD is often mentioned together,  
giving the impression it's a single undivided system.

### But in GitOps era, it's easier to split as:

#### CI Focuses More On:

- Pull code
- Testing
- Build
- Push Harbor

That is:

**Transform source code into image artifacts**

#### CD Focuses More On:

- Update deployment configuration in Git
- Let Argo CD synchronize cluster
- Let cluster continuously align with target state

That is:

**Stabilize artifacts and configuration in environments**

### Current Understanding to Establish

This is why some teams say:

> GitOps isn't replacing CI, but redefining CD implementation.

---

## Part 9: What's the Best "Boundary Definition" for Your Current Context

Recommend fixing the following definition first, as it's very useful for future work.

### GitLab CI/Jenkins

More focused on:

- Build and artifact production

### Harbor

More focused on:

- Image storage and governance

### Helm

More focused on:

- Configuration templating and parameterization

### Argo CD

More focused on:

- Continuous alignment of Git target state

### kubectl

More focused on:

- Observation, troubleshooting, and basic operations

### Current Understanding to Establish

As long as you clearly define these boundaries,  
you'll avoid confusion in interviews or discussions about solutions.

---

## Part 10: Understanding Minimum Migration Path from Traditional to GitOps

This section is highly practical.

Currently, we won't discuss large-scale platform migration, only the minimum migration path you can understand now.

### Step 1: Keep Existing CI Build Chain

That is:

- Code
- Build
- Tag
- Push Harbor

Keep this unchanged for now.

### Step 2: Consolidate Deployment Configuration into Git

For example:

- Helm Chart
- values-dev.yaml
- values-test.yaml

You've already done this earlier.

### Step 3: Reduce Direct Manual Cluster Modifications

Replace:

- `kubectl set image`
- Manual `helm upgrade`

Gradually with:

- Modify values in Git
- Let Argo CD synchronize

### Step 4: Pilot in Non-Production Environments First

For example:

- Start with dev/test first
- Evaluate prod later

### Current Understanding to Establish

GitOps migration isn't "rebuilding everything from scratch,"  
but rather:

**Preserve existing CI capabilities, gradually shifting the release entry point to Git.**

---

## Part 11: A Minimum Migration Example Based on Your Current Context

You've already done:

### Old Way

1. Build image
2. Push Harbor
3. Manually execute:
       helm upgrade manual-web-test . -f values-test.yaml -n test

### New Way

1. Build image
2. Push Harbor
3. Modify Git repository's:
       manual-web/values-test.yaml
4. Commit + push
5. Argo CD detects OutOfSync
6. Sync
7. Cluster updates

### Current Understanding to Establish

This is the minimum migration difference:

- CI front half remains unchanged
- Helm template remains unchanged
- Harbor remains unchanged
- The real change is the "release entry point"

---

## Part 12: When Is It More Suitable to Continue Using Traditional Push-Based Releases

This point needs to be clearly explained, otherwise it might give the impression that "GitOps is absolutely superior."

Current scenarios where traditional methods are more suitable typically include:

### Scenario 1: Few Environments, Small Team

- Just one or two environments
- Maintained by one or two people
- Low release frequency

### Scenario 2: Main Issues Are in the Front Half

For example:

- Builds are still unstable
- Harbor hasn't been organized
- Helm hasn't been standardized

In such cases, adopting GitOps too early could add complexity.

### Scenario 3: Temporary Experiments, Short-Term Demonstration Environments

- Pursue rapid experimentation
- Not necessarily require long-term state tracking

### Current Understanding to Establish at This Step

GitOps is not "something that must be implemented immediately in all scenarios",  
but rather when:

- Environments become numerous
- Collaboration becomes complex
- State tracking requirements increase

The value becomes increasingly apparent.

---

## Part 13: When It's More Suitable to Introduce Argo CD / GitOps Gradually

The typical scenarios suitable for introducing GitOps at this stage include:

### Scenario 1: Multi-Environment Differences Become Obvious

For example:

- dev
- test
- prod

And each environment has clear differences in values.

### Scenario 2: Manual Cluster Modifications Become Too Frequent, Leading to Chaos

- Someone directly modifies Deployment
- Git and the actual cluster become increasingly out of sync
- State tracking becomes difficult

### Scenario 3: Want to Turn "Deployment Actions" into "Configuration Review Actions"

That is:

- Not directly modifying the cluster
- But first modifying Git
- Making changes more reviewable and traceable

### Scenario 4: Start Needing Drift Detection and Continuous Alignment

This is exactly what Argo CD excels at.

### Current Understanding to Establish at This Step

What makes GitOps most attractive is not just "automation",  
but:

**Making environment states more trackable, reviewable, and recoverable.**

---

## Part 14: Recommended Final Understanding Approach at This Stage

It's recommended to fix this entire relationship into the followingcaliber.

### CI Does Not Disappear

Still responsible for:

- build
- test
- push Harbor

### Helm Does Not Disappear

Still responsible for:

- Chart
- templates
- values

### Harbor Does Not Disappear

Still responsible for:

- Image storage
- tag governance
- scanning / replication / Webhook capabilities, etc.

### Argo CD's Core Value

Responsible for:

- Continuously aligning the declared configuration state in Git to K8s

### Current Understanding to Establish at This Step

So a truly mature system is not "only Argo CD",  
but:

**CI, Harbor, Helm, and Argo CD each doing what they're best at.**

---

## Part 15: Express the Entire Chain in One Complete Sentence

At this point, you should already be able to explain the entire main line clearly.

It's recommended to try restating it with the followingcaliber:

After application code changes, first test, build, and package images through GitLab CI or Jenkins, then push the images to Harbor.  
For deployment configuration, express different environment target states through Helm Chart and values files.  
If following traditional deployment, subsequent manual or pipeline execution of `kubectl` or `helm upgrade` will push changes to the cluster; if following GitOps, Argo CD will read the target state configuration from Git, continuously compare and synchronize cluster states.  
Therefore, traditional CI/CD and Argo CD are not mutually exclusive, but rather each excels at different parts of the delivery pipeline.

### Current Understanding to Establish at This Step

If you can clearly explain this, it means the main line of this stage is truly connected.

---

## Part 16: This Section's Mini Exercise

### Exercise 1: Draw a Comparison Diagram Between "Traditional Deployment" and "GitOps Deployment"

Requirements: At least draw these nodes:

- Code repository
- CI (GitLab CI / Jenkins)
- Harbor
- Helm / values
- kubectl / helm upgrade
- Git
- Argo CD
- K8s Cluster

And label:

- Which are part of the first half
- Which are part of the second half
- Where the differences in deployment entry points lie

### Exercise 2: Write Both Deployment Processes Using Your Current `manual-web` Main Line

#### Write a Traditional Process

For example:

- build
- push
- helm upgrade

#### Write a GitOps Process

For example:

- build
- push
- modify values
- commit + push
- Argo CD sync

### Exercise 3: Answer the Following 6 Questions Yourself

1. Why do we still need Argo CD if we have GitLab CI / Jenkins?
2. Do we still need CI if we have Argo CD?
3. What roles does Helm play in both modes?
4. What does it mean to "shift the deployment entry point from command-line to Git target state"?
5. Why do many teams coexist with both modes long-term?
6. What's the minimal approach to migrate to GitOps at this stage?

If you can answer these 6 questions, you've mastered this section.

---

## Content to Be Able to Explain After This Section

After completing this section, it's recommended to be able to clearly explain the following:

Traditional CI/CD and Argo CD are not mutually exclusive, but rather each excels at different parts of the delivery pipeline.  
GitLab CI and Jenkins are more focused on the first half, responsible for testing, building, tagging, and pushing to Harbor, transforming code into deployable artifacts; Helm is more focused on configuration templating and parameterization; Argo CD is more focused on the second half, responsible for reading the target state from Git and continuously aligning it to the Kubernetes cluster.  
Therefore, the value of GitOps is not to replace all previous tools, but to gradually shift the "deployment configuration and environment state" part from manual command-driven to Git-driven and controller-aligned.  
In real teams, these two modes often coexist for a long time, gradually completing the migration rather than replacing everything at once.

## Common Issues and Troubleshooting Directions

### Issue 1: Why does it feel like there are more tools at the end?

Because what you're seeing isn't "redundant tools", but different layers of capability along a complete delivery chain:

- Build
- Repository
- Templates
- Deployment
- State alignment

### Issue 2: Is GitOps necessarily more advanced than traditional deployment?

No, don't understand it that simply.  
More accurately:

- It's more suitable for scenarios with multiple environments, multi-person collaboration, and higher state tracking requirements

### Issue 3: If I haven't fully mastered Helm yet, should I delay Argo CD?

Yes.  
Because the clearer you are with Helm and values, the smoother the transition to Argo CD will be.

---

## Key Takeaways from This Section

1. The boundary between traditional CI/CD and Argo CD
2. The position of CI, Harbor, Helm, and Argo CD in the entire chain
3. The reDivision of labour of CI/CD in the GitOps era
4. Why two modes often coexist for a long time
5. The basic approach for minimal migration to GitOps

## One-Sentence Summary

Argo CD is not here to replace GitLab CI, Jenkins, Harbor, and Helm, but to further incorporate their prepared images and configurations into a Git-driven and continuously aligned deployment system.

1. Understand why CI/CD is needed in Kubernetes scenarios  
2. Understand the minimal release pipeline stages  
3. Understand how images are built and why tags are important  
4. Understand Harbor's role in the image repository  
5. Understand the minimal structure of GitLab CI/Jenkins Pipeline  
6. Understand the executor perspective of Runner/Agent/Jenkins on Kubernetes  
7. Understand Deployment rolling updates, release validation, and rollback  
8. Understand Helm's templating, values, and multi-environment overrides  
9. Understand CI/CD security and Harbor advanced governance  
10. Understand blue-green, canary, and gray release strategies  
11. Understand the segmented troubleshooting approach for standard CI/CD  
12. Understand GitOps' target state concept  
13. Understand Argo CD installation, objects, Helm integration, Git repository-driven release, and troubleshooting  
14. Finally understand the boundaries andDivision of labour between Argo CD and traditional CI/CD  

### Current understanding to establish  

This is already a complete "from image to cluster, from manual release to GitOps" main line.  

---  

## II. Recommended way to explain "minimal delivery chain" now  

You can try to restate it with the following paragraph:  

In Kubernetes scenarios, application delivery typically starts with code or content changes, first building images via Dockerfile, then using appropriate tags for versioning, and pushing to Harbor as a unified image repository.  
Configuration layers are usually expressed through YAML or Helm Chart + values to represent target states for different environments.  
If using traditional methods, GitLab CI or Jenkins can execute kubectl or helm commands to push changes to the cluster after build and push. If using GitOps, Argo CD continuously reads configuration target states from Git and synchronizes them to Kubernetes.  
After deployment, you need to monitor rollout, Pods, Services, and business results, and have rollback and troubleshooting capabilities.  

If you can smoothly explain this, it means this stage has truly formed a system.  

---  

## III. The most valuable actions to repeatedly practice at this stage  

Not all knowledge points need equal repetition. Currently, the most valuable actions to repeatedly practice are the following.  

### 1) Build / Tag / Push  

That is:  

- `docker build`  
- `docker tag`  
- `docker push`  

### 2) Deployment updates and rollout observation  

That is:  

- `kubectl set image`  
- `kubectl rollout status`  
- `kubectl describe pod`  

### 3) Helm values changes and upgrade  

That is:  

- `helm template`  
- `helm upgrade`  
- `helm history`  
- `helm rollback`  

### 4) Argo CD minimal GitOps change chain  

That is:  

- Change `values-test.yaml`  
- commit + push  
- Argo CD OutOfSync  
- Sync  
- Business validation  

### 5) Troubleshooting sequence  

That is breaking down the problem into:  

- Git / Configuration  
- Build  
- Push  
- Sync / Deploy  
- Pod / Rollout  
- Business results  

---  

## IV. The biggest change you'll have after completing this stage  

Initially, you were more like:  

- Only knew a few isolated commands  
- Only knew the names of Jenkins / GitLab CI / Harbor / Helm / Argo CD  
- But didn't understand how they connected  

Now, you should already be able to:  

### 1) Know how the entire chain connects  

Not viewing tools in isolation, but knowing:  

- Code  
- Image  
- Repository  
- Configuration  
- Cluster  
- Controller  

how they are connected.  

### 2) Know what each tool does separately  

You won't confuse:  

- CI  
- Harbor  
- Helm  
- Argo CD  

### 3) Know where the problem lies in which segment  

This is a significant progress.  
More valuable than "remembering more commands".  

### 4) Begin to have a "platform perspective"  

This means you're not just deploying a single application,  
but starting to understand:  

- How the release system is designed  

---  

## V. The natural next step after completing this stage  

If you continue to deepen, the most natural next step is usually entering the following directions.  

### Direction 1: Turn the current main line into a more realistic project practice  

For example:  

- Full chain of frontend/API/Nginx/ConfigMap/Ingress  
- Truly connect to GitLab CI/Jenkins  
- Truly connect to Argo CD automatic synchronization  

### Direction 2: Complement K8s release surrounding capabilities  

For example:  

- Ingress / domain / TLS  
- ConfigMap / Secret release strategies  
- HPA / PDB / Readiness / Liveness impact on release  
- Resource limits and release stability  

### Direction 3: Deepen GitOps further  

For example:  

- Application, Project, Repository details  
- Sync / Diff / Health / Self-Heal  
- App of Apps  
- Multi-environment Git repository design  

### Direction 4: Deepen platform and security further  

For example:  

- Credential governance  
- Image scanning and admission  
- Release approval  
- Multi-cluster distribution  
- Release audit and traceability  

---  

## VI. Final one-sentence summary for this stage  

From 01 to 28, this stage truly establishes not just several release commands, but the entire minimal cloud-native delivery main line from image building, repository storage, configuration templating, to traditional release, GitOps alignment, rollback, and troubleshooting.