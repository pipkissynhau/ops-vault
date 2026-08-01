# 18-Harbor Advanced: Replication, Scanning, Webhook, and Project Governance

## Document Notes

This is the 18th note in the 08-CI-CD learning path.

Previously, we've covered the basics of Harbor:

- Harbor is a unified image repository
- What projects, repositories, and tags are
- Robot accounts are suitable for CI pipelines
- Kubernetes needs to consider authentication and trust when pulling private images from Harbor

This article continues the progression, but focuses not on "large concept explanations" but on four directions you can directly observe and operate:

1. Tag and repository governance
2. Image replication concepts
3. Vulnerability scanning entry points
4. Webhook triggering logic

This article still aligns with your current environment:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private repository
- `test` namespace
- Existing `manual-web` repository with multiple tag groups

## Tags

#Kubernetes #CI-CD #Harbor #MirrorCopy #MirrorScan #Webhook #ProjectGovernance #MirrorRepository #I'llTakeYourNotes.

## Learning Objectives

After completing this article, you should be able to:

1. Understand tags within the same repository in Harbor
2. Recognize what Harbor project governance should prioritize first
3. Understand when image replication is appropriate
4. Locate Harbor's vulnerability scanning entry point and understand how to use the results
5. Understand where Webhook fits in the release pipeline
6. Explain the relationship between Harbor's advanced capabilities and its current mainline features

## This Article's Experiment Focus

This article doesn't require you to set up all Harbor's advanced capabilities today.  
Currently, focus on 4 key tasks:

1. Conduct a repository and tag governance observation using the images you've already pushed
2. Explore Harbor's scanning capabilities and understand "images also need inspection"
3. Examine replication and Webhook locations to establish usage scenario awareness
4. Combine with the current `manual-web` repository to summarize a minimal governance rule

---

## Part 1: First, organize the tags in the current `manual-web` repository

This section is important because the first step in Harbor's advanced features isn't replication or scanning, but:

**First, make sure the tags in the repository are well-organized.**

### Step 1: Confirm existing tags locally

Run:

    docker images | grep manual-web

You likely have similar tags locally:

- `v10`
- `v11`
- `latest`
- `dev-e8f7a6b-1101`
- `dev-13`
- `test-13`

### Step 2: Enter the repository in Harbor

Navigate to:

- Project: `test`
- Repository: `manual-web`

Record all current tags.

### Step 3: Manual grouping

Suggest you create a minimal grouping record, for example:

#### Learning tags
- `v10`
- `v11`

#### Environment tags
- `dev-13`
- `test-13`

#### Production understanding tags
- `dev-e8f7a6b-1101`

#### Temporary tags
- `latest`

### Understanding to establish at this stage

The first step in Harbor's advanced governance isn't talking about platform-wide capabilities,  
but:

**Give clear semantics to tags within the same repository.**

If this layer is chaotic, it will be difficult to effectively use scanning, replication, and Webhook later.

---

## Part 2: Practical check from a "repository organization perspective"

This section continues with the `manual-web` repository.

### Step 1: Identify currently used tags

Check which images the current Deployment is actually using:

    kubectl -n test describe deploy manual-web | grep -A3 Image

If you still have a dev environment, you can also check:

    kubectl -n dev describe deploy manual-web | grep -A3 Image

### Step 2: Compare with Harbor repository page

Determine from the Harbor page:

- Which tags are currently referenced by the environment
- Which tags are historical experiment leftovers
- Which tags are "meaningful to retain"
- Which tags are "can be cleaned up later"

### Understanding to establish at this stage

The first thing project governance should do isn't complex rules,  
but first distinguish:

- Currently in use
- Historical versions
- Temporary experiments

This step you can do now, and it's very valuable.

---

## Part 3: What should Harbor project governance focus on first in the current stage

Don't talk about comprehensive governance systems yet. Focus on the 4 most useful things first.

### 1) Don't mess up repository naming

You already have:

- `manual-web`

This is good.  
Keep the same application in the same repository for as long as possible, don't change names frequently.

### 2) Don't mess up tag semantics

For example:

- Learning tags: `v10`
- Environment tags: `dev-13`
- Production understanding tags: `dev-e8f7a6b-1101`

As long as you can understand at a glance, it's progress.

### 3) Don't accumulate meaningless tags long-term

If some tags are for temporary testing, you should have a cleanup mindset later.

### 4) Try to unify image entry points

Regardless of manual deployment, Helm, GitLab CI, Jenkins, try to follow the same project/repository/tag rules in Harbor.

### Understanding to establish at this stage

Harbor governance isn't about complex systems from the start,  
but first organize:

- Naming
- Tags
- Usage status

These three things.

---

## Part 4: Hands-on - Push a group of "more standardized" tags to feel the governance effect

To make this article more concrete, suggest you push a group of clearer tags.

Assume the current content is prepared as a new version `v18`.

### Step 1: Prepare new content

Enter the application directory:

    cd ~/08-ci-cd/01-manual-release

Modify the page:

    cat > index.html <<'EOF'
    <html>
      <head>
        <meta charset="utf-8">
        <title>manual release v18</title>
      </head>
      <body>
        <h1>Hello K8s CI/CD</h1>
        <p>version: v18</p>
      </body>
    </html>
    EOF

### Step 2: Build local image

    docker build -t manual-web:v18 .

### Step 3: Add three types of tags

#### Learning tags /think
</think>

#### Environment tags
- `dev`
- `f1e2d3c`

#### Production understanding tags
- `1801`

### Step 4: Push to Harbor

    docker push `test`

### Step 5: Observe governance effects

After pushing, check:

1. Are the new tags clearly organized?
2. Can you quickly identify which tags are in use?
3. Do the tags reflect their intended purpose?

This exercise will help you see how governance improves clarity and maintainability.

docker tag manual-web:v18 manual-web:v18

This step actually already exists during the build process.

#### Environment-type tag

    docker tag manual-web:v18 manual-web:test-18

#### Production-understanding tag

Assume:

- Branch: `dev`
- Commit: `f1e2d3c`
- Pipeline: `1801`

Execute:

    docker tag manual-web:v18 manual-web:dev-f1e2d3c-1801

### Fourth Step: Push to Harbor

    docker tag manual-web:v18 your-Harbor-domain/test/manual-web:v18
    docker push your-Harbor-domain/test/manual-web:v18

    docker tag manual-web:v18 your-Harbor-domain/test/manual-web:test-18
    docker push your-Harbor-domain/test/manual-web:test-18

    docker tag manual-web:v18 your-Harbor-domain/test/manual-web:dev-f1e2d3c-1801
    docker push your-Harbor-domain/test/manual-web:dev-f1e2d3c-1801

### Fifth Step: Check Effect on Harbor Page

You will now see more regular tag patterns in the same repository.

### Current Understanding to Establish at This Step

As long as the tag strategy is clear, the Harbor page itself will be easier to understand than before, which was "a pile of random tags."

This is the most direct benefit of Harbor governance.

---

## Fifth Part: What is Image Replication, and How to Understand It

This part does not require you to set up replication rules immediately. First, establish the correct usage scenarios.

Harbor's replication (Replication) capability primarily solves:

**Synchronizing images from one Harbor or repository location to another.**

### Common Use Cases

#### Scenario 1: Synchronization Between Different Data Centers / Repositories

For example:

- Harbor in Data Center A
- Harbor in Data Center B

Desire to replicate images to the other location.

#### Scenario 2: Synchronization from Test Repository to Production Repository

For example, first build/push in a project, then synchronize to a more formal project.

#### Scenario 3: Distribution to Edge / Offline Environments

Images cannot be pulled from the central repository every time, so they need to be pre-replicated.

### Current Understanding to Establish at This Step

Replication is not "rebuilding the image,"  
but rather:

**Synchronizing existing image versions at the repository level.**

---

## Sixth Part: Finding Replication-Related Entries on the Harbor Page

This part focuses primarily on "viewing entries and building a sense."

### First Step: Enter the Harbor Page

Check the left sidebar or project management area for similar entries:

- Replication
- Registries
- Replication Rules

Harbor page terminology may vary slightly by version, but similar locations are generally available.

### Second Step: Observe First, Do Not Rush to Configure

First glance:

- Replication rules typically require a source and target
- Need to select project/repository scope
- Need to define trigger methods

### Current Understanding to Establish at This Step

You may not have a dual Harbor scenario immediately, but you should know:

**Replication is Harbor's "image distribution/synchronization capability," not the publishing command itself.**

---

## Seventh Part: What is Image Scanning, and How to Understand It

The core of image scanning (Scan) is answering:

**Does this image contain known vulnerability risks?**

This directly connects to the "artifact security" discussed in your 17th article earlier.

### Why Scanning is Important

Because images are not neutral files; they eventually run in clusters.  
If the base image has high-risk vulnerabilities or certain package versions have known risks, you at least need to know.

### Current Understanding to Establish at This Step

Scanning is not about achieving perfect security from the start,  
but rather about beginning to establish:

> Pushing an image does not mean it's automatically "trusted and usable."

---

## Eighth Part: Finding the Scan Entry on the Harbor Page

This part recommends you directly interact with the interface.

### First Step: Enter the Repository

Enter:

- Project: `test`
- Repository: `manual-web`

### Second Step: Click a Specific Tag

For example:

- `v18`
- Or `dev-f1e2d3c-1801`

Check if the detail page has similar entries or information:

- Scan
- Vulnerabilities
- Scan Results
- Critical/High/Medium/Low vulnerability counts

UI may vary by Harbor version, but scanning-related locations are generally available.

### Third Step: If the Page Supports Manual Scanning

Try scanning a tag.

### Current Understanding to Establish at This Step

At this stage, you don't need to overly pursue "scan results must be all 0."  
Just do these two things:

1. Know where the scan entry is
2. Understand that scan results provide reference for release decisions

---

## Ninth Part: How to Read Minimum Scan Results

If the scan function is available, you'll typically see similar:

- Critical
- High
- Medium
- Low

### Most Appropriate Understanding at This Stage

#### 1) Focus First on Obvious High-Risk Issues

It's not about rejecting learning entirely if there are vulnerabilities, but at least know:

- Is the current image clearly at high risk

#### 2) Then Review the Base Image Source

For example, your commonly used base images are:

- `nginx:1.27`
- `busybox:1.35`
- `alpine:3.20`

If scan results are poor, the first reaction is usually not "Harbor is broken,"  
but rather:

- Base image version
- Software packages included in the image

### Current Understanding to Establish at This Step

The value of scan results starts with "seeing problems,"  
then proceeds to subsequent governance steps.

---

## Tenth Part: What is a Webhook, and How to Understand It

In Harbor, a webhook can be initially understood as:

**After certain events occur in a repository, actively sending a notification to an external system.**

For example:

- New image pushed successfully
- Image deleted
- Certain project events occur

Then Harbor sends this event to a specific URL.

### Why It's Useful

Because you may later want:

- Harbor to notify a system after receiving a new image
- Trigger an automated action
- Let external platforms sense repository changes

### Current Understanding to Establish at This Step

Webhooks are not build commands or publish commands,  
they are more like:

**The "outward notification capability" for repository events.**

---

## Eleventh Part: Finding Webhook Entry on the Harbor Page

### First Step: Enter the Project-Level Management Location

Enter project `test`, check if there are similar entries:

- Webhooks
- Notifications
- Event

### Second Step: Observe Configuration Items /think

Usually you'll see:

- Target URL
- Trigger event type
- Enabled/disabled status

### Understanding to establish at this stage

You may not currently have a suitable external URL to actually receive notifications,  
but you need to first understand:

**Webhook is one of the connection points between Harbor and external automation systems.**

---

## Part 12: How to best understand Webhook in the current mainline

Combined with what you've already learned in the mainline, the most natural understanding is:

### Active actions in the current mainline

- Build image
- Push to Harbor
- kubectl / Helm deployment

### Location of Harbor Webhook

It is not a replacement for these actions, but rather occurs:

- After the image enters Harbor
- To notify "what happened in the repository" externally

So you can think of it as:

**The bridge between Harbor events and external systems.**

For example, Webhook will only truly take effect when you have more complex platforms later.

---

## Part 13: Harbor governance habits most worth establishing at this stage

This section directly gives you a set of practical habits for the current stage.

### Habit 1: First clarify the semantic meaning of tags

At least you should be able to distinguish at a glance:

- Learning type
- Environment type
- Production understanding type

### Habit 2: After pushing, always confirm on Harbor page

Don't just rely on command line success, confirm:

- Correct project
- Correct repository
- Correct tag

### Habit 3: Start paying attention to scan entry points

Even if you don't implement strict gatekeeping now, first establish the habit of "checking image risk results".

### Habit 4: Know the location of replication and Webhook, don't rush to overuse

At this stage, knowing "what capabilities exist" and "what scenarios are suitable" is enough.

---

## Part 14: Exercises in this section

### Exercise 1: Organize tags in `manual-web` repository

Requirements:

- Go to Harbor page and classify all current tags in `manual-web` into 3 categories
- Mark out:
  - In-use
  - Historical retention
  - Temporary experimentation

### Exercise 2: Push a group of more standardized tags

At least push:

- A learning-type tag, e.g. `v18`
- An environment-type tag, e.g. `test-18`
- A production understanding-type tag, e.g. `dev-f1e2d3c-1801`

### Exercise 3: Manually check the scan entry point once

Requirements:

- Enter a tag's details
- Find the scan location
- Record which risk level fields you saw

### Exercise 4: Answer the following 5 questions yourself

1. What should Harbor project governance prioritize first?
2. What problems does image replication solve?
3. Why is image scanning related to the release pipeline?
4. What role does Webhook play in Harbor?
5. Why is Harbor's advanced capabilities not "better to enable all as early as possible", but rather to first organize tags and repositories?

If you can explain these 5 questions yourself, you've mastered this section.

---

## Content you should be able to explain after this section

After completing this section, it's recommended you can clearly explain the following:

Harbor's advanced capabilities aren't about enabling all features from the start, but rather starting with repository governance.  
For tags in the same repository, first distinguish between learning-type, environment-type, and production understanding-type tags, and clearly identify which tags are currently in use by environments.  
Image replication solves image synchronization issues between repositories or environments, image scanning addresses "whether the image content has known risks", and Webhook handles how repository events notify external systems.  
Therefore, Harbor's advanced capabilities essentially build upon "image storage and retrieval" to further enhance governance, distribution, inspection, and integration capabilities.

## Common issues and troubleshooting directions

### Issue 1: Why does my Harbor repository seem to be getting more chaotic

Usually it's not Harbor itself that's chaotic, but:

- Tag semantics not clearly defined
- Not distinguishing between in-use and historical tags
- No cleanup awareness after pushing

### Issue 2: Is it necessary to have all green scan results to allow learning?

No.  
At this stage, the focus is on learning "how to view scan entry points and understand the meaning of scan results".

### Issue 3: Since replication and Webhook don't currently connect to real Harbor/external systems, is this section unnecessary?

It is necessary.  
At this stage, first establish awareness of capability boundaries, so you won't be unfamiliar when encountering more complex scenarios later.

---

## Key takeaways of this section

1. Minimum starting point for Harbor repository governance
2. Tag grouping and usage status identification
3. Applicable scenarios for image replication
4. Scan entry point and basic understanding
5. Webhook's role in Harbor

## One-sentence summary

Harbor's advancement isn't about enabling all advanced features upfront, but rather first clearly organizing repositories and tags, then gradually introducing replication, scanning, and Webhook capabilities to evolve image repositories from "able to store images" to "able to govern images".