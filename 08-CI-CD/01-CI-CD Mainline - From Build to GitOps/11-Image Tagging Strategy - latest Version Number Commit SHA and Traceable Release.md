# 11-Image Tagging Strategy: latest, Version Numbers, Commit SHA, and Traceable Releases

## Document Notes

This is the 11th note in the 08-CI-CD learning path.

The previous 01-10 sections have already established the minimal release pipeline, and this note begins to address "production-level details".  
The first essential detail to complete is the **image tagging strategy**.

Image tagging is not a minor detail; it directly affects:

- Release tracking
- Rollback speed
- Fault localization
- Multi-environment collaboration
- Pipeline maintainability

This note continues to align with the current experimental environment:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private registry
- `test` namespace

## Tags

#Kubernetes #CI-CD #Docker #Harbor #MirrorTag #latest #commitsha #pipelineid #VersionManagement #I'llTakeYourNotes.

## Learning Objectives

After completing this note, you should be able to:

1. Understand the role and limitations of `latest`
2. Understand what problems version numbers, commit SHA, and pipeline ID solve respectively
3. Design an image tagging strategy suitable for the current stage
4. Maintain both "learning tags" and "production tags" in Harbor
5. Quickly determine the origin of an image through tags
6. Explain why it's not advisable to use `latest` long-term in production

## Main Experiment Flow for This Note

This note does not introduce new tools and continues to use your existing:

- Local builds
- Harbor pushes
- K8s Deployment updates
- Cluster access verification

This note focuses on three main tasks:

1. Compare the significance of different tags
2. Manually assign multiple tags to the same image
3. Release using different tags and observe why "traceable tags" are better for subsequent CI/CD

---

## Part 1: First Answer a Core Question - Why Must Image Tags Be Discussed Separately

You've already used:

- `v1`
- `v2`
- `v3`
- `dev-a1b2c3d-101`

If you're just doing a few experiments, these names can work.  
But once release actions increase, you'll start encountering these issues:

- Too many tags in Harbor, unsure which is the official one
- Can't tell which version a Deployment is using at a glance
- Don't know which tag to rollback to when problems occur
- Mirror sources become increasingly ambiguous after repeated builds on the same branch
- Can't quickly map pipeline-built images to specific code and build tasks

So from this note onward, "just writing a tag" should be upgraded to "designing a rule-based version identity".

---

## Part 2: Understand the Role of Tags with One Sentence

The core role of image tags is:

**To give images an identifiable, traceable, and rollable version identity.**

So tags aren't for aesthetics or just naming.  
They are essentially version anchors in the release pipeline.

---

## Part 3: Common Tag Strategies

This section first lists common forms without judgment.

### 1) latest

Example:

    manual-web:latest

Features:

- Simplest
- Easiest to remember
- Most common in local testing

Issues:

- Can't see source
- Can't see branch
- Can't see code commit
- Can't see which pipeline build
- Not conducive to rollback
- Not helpful for troubleshooting

Use Cases:

- Local temporary verification
- Early learning phase quick experiments

Not Suitable For:

- Long-term official releases
- Multi-person collaboration environments
- Production environments

---

### 2) Manual Version Number

Example:

    manual-web:v1
    manual-web:v2
    manual-web:v3
    manual-web:v11

Features:

- Intuitive
- Suitable for observing version changes in the learning phase
- Easy for human eyes to understand

Issues:

- Can't see code source
- Can't see which branch it comes from
- Can't see which build it belongs to
- Prone to confusion with manual maintenance

Use Cases:

- Current learning phase
- Demo version evolution
- Minimal manual release pipeline experiments

---

### 3) Branch Name

Example:

    manual-web:dev
    manual-web:main
    manual-web:test

Features:

- Can immediately see which development line the image comes from

Issues:

- Same branch will overwrite repeatedly
- Insufficient precision
- Still not suitable for official version tracing

Use Cases:

- Temporary environments
- Simple differentiation between development lines

---

### 4) Branch Name + Commit SHA

Example:

    manual-web:dev-a1b2c3d

Features:

- Can locate to specific code commit
- Clearly enhanced traceability
- Clearer rollback

Issues:

- If the same commit runs the pipeline multiple times, it's still hard to distinguish specific builds

Use Cases:

- Already integrated with Git repository
- Want images to correspond one-to-one with code commits

---

### 5) Branch Name + Commit SHA + Pipeline ID

Example:

    manual-web:dev-a1b2c3d-101

Features:

- Can see branch
- Can see code commit
- Can see specific pipeline task
- Most suitable for correspondence with GitLab CI/Jenkins later

Issues:

- Longer name
- Less intuitive for beginners compared to `v1/v2`

Use Cases:

- Closer to production
- Pipeline builds
- Need for quick image source tracing

---

## Part 4: Why Production Shouldn't Rely Solely on latest Long-Term

This point must be thoroughly explained as it's the most common misconception.

### Problem 1: Can't Identify Image Source

If a Deployment writes:

    image: harbor.example.com/test/manual-web:latest

You can only know it "looks like the latest", but you don't know:

- Which branch it comes from
- Which commit it corresponds to
- Which build it belongs to
- Who pushed it up

---

### Problem 2: Difficult Rollback

If there's a problem in production, the most important thing is:

- Roll back to the previous stable image

But if the registry only maintains latest long-term, you'll struggle to quickly determine:

- Which version was the previous stable one
- How many times this latest has overwritten
- Which one was the stable version

---

### Problem 3: Troubleshooting Difficulties

Often, the first step in troubleshooting is:

- Determine which code change the current Pod's image corresponds to

If only latest is used, you'll need to check many places extra, making the localization process inefficient.

---

### Problem 4: Same Tag Overwrite Can Create Illusions

For example:

- You rebuild with new content
- Still push to latest
- K8s Deployment still writes latest

You'll think "I've released the new version", but the nodes may not have picked up the new content as intended.

So:

**latest isn't unusable, but it shouldn't be the sole official version identifier long-term.**

---

## Part 5: Why Commit SHA Is Important

Commit SHA solves:

**The correspondence between images and code commits.**

For example: /think

manual-web:dev-a1b2c3d

This tag at least tells you:

- This image comes from the dev branch
- The corresponding commit is `a1b2c3d`

This allows you to quickly locate in the code repository during troubleshooting:

- What changes were made in this version

This is extremely valuable in production.

---

## Part Six: Why Pipeline ID is Also Important

Pipeline ID resolves:

**The correspondence between images and specific build tasks.**

Assume a commit:

- First build failed
- Second run after environment fix
- Third triggered manually

These three may all point to the same commit, but they may not all produce the same final image.

So if the tag only writes:

    dev-a1b2c3d

You can only know which commit it comes from, but cannot quickly determine:

- Which specific build task it came from

If you add pipeline ID:

    dev-a1b2c3d-101

You can further locate:

- This is the image built by pipeline 101

This is very helpful for combining with GitLab CI / Jenkins to check logs.

---

## Part Seven: Recommended Tag Usage at This Stage

Based on your current learning state, we recommend using two layers.

### First Layer: Learning Tags

Keep:

- `v1`
- `v2`
- `v3`
- `v10`
- `v11`

Purpose:

- Facilitate observing version changes
- The relationship between page content and image versions is most intuitive

### Second Layer: Production Understanding Tags

Keep synchronized:

- `dev-a1b2c3d-101`
- `main-f6e7d8c-205`
- `dev-e8f7a6b-1101`

Purpose:

- Establish a production perspective
- Prepare for connecting with GitLab CI / Jenkins later
- Train your ability to directly determine the source from the image

### Understanding to Establish at This Stage

At this stage, it's not a choice between two options, but rather:

**Keep both learning tags and production understanding tags for the same image.**

This way, you won't be hindered by long tag names during the learning phase, while gradually adapting to production thinking.

---

## Part Eight: Hands-on - Tagging the Same Image with Multiple Tags

This section directly performs the action.

Assume you already have a local image:

    manual-web:v10

First check:

    docker images | grep manual-web

### Step 1: Tag the Same Image with a latest

Execute:

    docker tag manual-web:v10 manual-web:latest

### Step 2: Tag the Same Image with a Production Understanding Tag

Assume this build comes from:

- Branch: `dev`
- Commit: `e8f7a6b`
- Pipeline: `1101`

Execute:

    docker tag manual-web:v10 manual-web:dev-e8f7a6b-1101

### Step 3: Check Local Images

Execute:

    docker images | grep manual-web

### Expected Phenomenon

You'll see the same image content, now possibly with multiple tags, such as:

- `v10`
- `latest`
- `dev-e8f7a6b-1101`

### Understanding to Establish at This Stage

This shows:

- Tags are image reference names
- Multiple tags can point to the same image content
- The significance of tags lies in "how you identify it," not "the image content must be different"

---

## Part Nine: Hands-on - Pushing Multiple Tags to Harbor

Continue using the image from the previous section.

### Step 1: Push latest

Execute:

    docker tag manual-web:v10 your-Harbor-domain/test/manual-web:latest
    docker push your-Harbor-domain/test/manual-web:latest

### Step 2: Push Learning Version Number

Execute:

    docker tag manual-web:v10 your-Harbor-domain/test/manual-web:v10
    docker push your-Harbor-domain/test/manual-web:v10

### Step 3: Push Production Understanding Tag

Execute:

    docker tag manual-web:v10 your-Harbor-domain/test/manual-web:dev-e8f7a6b-1101
    docker push your-Harbor-domain/test/manual-web:dev-e8f7a6b-1101

### Step 4: Check on Harbor Page

Enter:

- Project: `test`
- Repository: `manual-web`

Confirm that the repository has at least these tags:

- `latest`
- `v10`
- `dev-e8f7a6b-1101`

### Understanding to Establish at This Stage

From Harbor's perspective, maintaining multiple tags in the same repository is normal.  
What's truly critical is:

- Which tag is used for learning observation
- Which tag is used for environment release
- Which tag is suitable as a tracking and rollback basis

---

## Part Ten: Hands-on - Updating Deployment with Different Tags

This section makes an important comparison.

### Scheme A: Deploy with Learning Tag

Execute:

    kubectl -n test set image deployment/manual-web manual-web=your-Harbor-domain/test/manual-web:v10

Check rollout:

    kubectl -n test rollout status deployment/manual-web

Verify inside the cluster:

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

After entering, execute:

    wget -qO- http://manual-web.test.svc.cluster.local

### Scheme B: Deploy with Production Understanding Tag

Execute:

    kubectl -n test set image deployment/manual-web manual-web=your-Harbor-domain/test/manual-web:dev-e8f7a6b-1101

Check rollout:

    kubectl -n test rollout status deployment/manual-web

Verify the page content again.

### Understanding to Establish at This Stage

From the page results, these two tags may reference the same content.  
But from the "version tracking" perspective, they differ greatly:

- `v10` is convenient for humans to visually track version evolution
- `dev-e8f7a6b-1101` is convenient for tracing image sources and build processes

This is why production environments prefer the latter.

## Part 11: Recommended Tag Rules for This Stage

This section suggests you remember these rules directly and use them as a guide.

### Rule 1: Do not treat 'latest' as the sole formal release identifier

It can exist, but long-term reliance on it is not recommended.

### Rule 2: During the learning phase, you can retain simple tags like v1/v2/v3/v10

Because it's most conducive to observing experimental results.

### Rule 3: For any image intended to be referenced by Harbor / Deployment / Helm values, try to apply production-grade semantic tags synchronously

Preferred format suggestion:

    branch-name-commitsha-pipelineid

Example:

    dev-e8f7a6b-1101

### Rule 4: Deployments and Helm values should reference explicit tags, not long-term rely on 'latest'

This makes releases more deterministic, and troubleshooting and rollback clearer.

---

## Part 12: Intentionally Create a "latest" Confusion Experiment

This experiment isn't to break the system, but to build intuition.

### Step 1: Prepare new content, e.g. v11-latest-test

    cd ~/08-ci-cd/01-manual-release

    cat > index.html <<'EOF'
    <html>
      <head>
        <meta charset="utf-8">
        <title>manual release latest test</title>
      </head>
      <body>
        <h1>Hello K8s CI/CD</h1>
        <p>version: v11-latest-test</p>
      </body>
    </html>
    EOF

### Step 2: Rebuild and overwrite 'latest'

    docker build -t manual-web:latest .
    docker tag manual-web:latest Yours.HarborDomain Name/test/manual-web:latest
    docker push Yours.HarborDomain Name/test/manual-web:latest

### Step 3: If your Deployment also long-term references 'latest'

You'll know 'latest' in Harbor has changed, but you'll remain ambiguous about "which version exactly".

### Current Understanding to Establish

The issue isn't that 'latest' can't be used, but that:

**It's unsuitable as a long-term trackable version identifier.**

You cannot determine from 'latest':

- Which code commit
- Which build
- Which release

---

## Part 13: This Section's Practice Exercises

### Exercise 1

Apply 3 tags to the current image simultaneously:

- `latest`
- `v11`
- `dev-abc1234-1102`

### Exercise 2

Push all 3 tags to Harbor and confirm on the page

### Exercise 3

Select any two tags, update Deployment respectively, and observe that business content remains consistent but "trackability" differs significantly

### Exercise 4

Answer these 4 questions yourself:

1. Why is 'latest' unsuitable as a long-term unique version identifier
2. What problem does commit sha solve
3. What problem does pipeline id solve
4. Why can learning environments retain v1/v2, but production environments recommend trackable tags

If you can explain these 4 questions, you've mastered this section.

---

## Content to Be Able to Explain After This Section

After completing this section, it's recommended to be able to explain this passage clearly:

The essence of image tags isn't about naming aesthetics, but giving images an identifiable, trackable, and rollable version identity.  
During the learning phase, you can first use simple tags like `v1/v2/v3` to observe version changes; but after entering production understanding, it's recommended to use trackable tags with branch names, commit shas, and pipeline ids, e.g. `dev-e8f7a6b-1101`.  
The benefit of such tags is being able to locate code sources and specific build tasks, making troubleshooting, rollback, and integration with GitLab CI/Jenkins smoother later.  
`latest` can be used for local quick experiments, but it's unsuitable as a long-term unique formal release identifier.

## Common Issues and Troubleshooting Directions

### Issue 1: Why can the same image have multiple tags

Because tags are essentially image reference names, and don't necessarily represent different image contents.

### Issue 2: Why pushing 'latest' isn't recommended long-term

Because the issue isn't whether pushing succeeds, but that it's unhelpful for tracking and rollback.

### Issue 3: Why a commit needs a pipeline id

Because the same commit may be built multiple times, and pipeline id helps locate specific build instances.

---

## Key Points to Master This Section

1. Core purpose of tags
2. Boundaries of 'latest'
3. Meaning of version numbers, commit shas, and pipeline ids
4. Appropriate dual-layer tag strategy for this stage
5. How to retain both learning and production tags simultaneously

## One-Sentence Summary

The essence of image tags isn't naming, but version tracking; during learning phases, simple version numbers can help understanding, but after entering production understanding, it's crucial to establish a trackable release mindset with "branch name + commit sha + pipeline id".