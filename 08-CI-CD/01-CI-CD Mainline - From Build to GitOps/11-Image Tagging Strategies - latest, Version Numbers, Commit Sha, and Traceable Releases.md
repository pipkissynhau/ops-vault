# 11-Image Tagging Strategies: `latest`, Version Numbers, Commit Sha, and Traceable Releases

## Document Description

This article is the 11th note in the 08-CI-CD learning pathway.

In previous sections 01-10, we have already established the minimum release pipeline. This article will focus on "production-level details."  
The first thing to address is **image tagging strategies**.

Image tagging is not a trivial matter; it directly affects:

- Release tracking
- Rollback speed
- Fault diagnosis
- Multi-environment collaboration
- Pipeline maintainability

This article continues to use the current experimental environment:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private repository
- `test` namespace

## Tags

#Kubernetes #CI-CD #Docker #Harbor #ImageTag #latest #commitsha #pipelineid #VersionManagement #PracticalNotes

## Learning Objectives

After completing this article, you should be able to:

1. Understand the role and limitations of `latest`.
2. Recognize what version numbers, commit sha, and pipeline id each solve.
3. Design a suitable image tagging strategy for your current phase.
4. Maintain both "learning tags" and "production tags" in Harbor.
5. Quickly identify the source of an image based on its tag.
6. Explain why using only `latest` is not recommended in production.

## Main Experimental Focus of This Article

No new tools will be introduced; we will continue to use what you already have:

- Local build
- Harbor deployment
- K8s Deployment updates
- In-cluster access verification

This article focuses on three main tasks:

1. Comparing the meanings of different tags.
2. Manually assigning multiple tags to the same image.
3. Deploying images with different tags and observing why "traceable tags" are more suitable for subsequent CI/CD processes.

---

## Part 1: Answer a Core Question—Why Should Image Tagging Be Discussed Separately?

You have already used labels like:

- `v1`
- `v2`
- `v3`
- `dev-a1b2c3d-101`

For a few experiments, these names are sufficient.  
However, as the number of releases increases, problems arise:

- Too many tags in Harbor, making it difficult to identify the official one.
- It's unclear which version is used by the Deployment.
- It's hard to determine which tag to roll back to when issues occur.
- Rebuilding from the same branch results in ambiguous image origins.
- Pipelines produce images that cannot be quickly linked back to specific code or build tasks.

Therefore, starting from this article, we will upgrade from "randomly adding tags" to "rule-based version identification."

---

## Part 2: Understand the Role of Tags in One Sentence

The core function of image tags is:

**To provide a recognizable, traceable, and rollable version identity for the image.**

Tags are not just for aesthetics or arbitrary names; they essentially serve as version anchors in the release process.

---

## Part 3: Common Tagging Strategies

In this section, we will list common forms without making judgments yet.

### 1) `latest`

Example:

    manual-web:latest

Characteristics:

- Simplest
- Easiest to remember
- Most commonly used for local testing

Problems:

- Unable to determine the source or branch.
- Cannot identify the code commit or build task.
- Difficult to use for rollbacks or troubleshooting.

Applicable scenarios:

- Temporary local verification
- Initial learning and quick experimentation

Not suitable for:

- Long-term official releases
- Multi-person collaboration environments
- Production scenarios

---

### 2) Manual Version Numbers

Example:

    manual-web:v1
    manual-web:v2
    manual-web:v3
    manual-web:v11

Characteristics:

- Intuitive
- Useful for observing version changes during learning
- Easy to understand visually

Problems:

- Unable to identify the code source or branch.
- Difficult to manage manually and prone to confusion.

Applicable scenarios:

- Current learning phase
- Demonstrating version evolution
- Simple manual release pipeline experiments

---

### 3) Branch Names

Example:

    manual-web:dev
    manual-web:main
    manual-web:test

Characteristics:

- Clearly indicate which development line the image comes from.

Problems:

- The same branch may be overwritten repeatedly.
- Lack of precision.
- Still not suitable for official version tracking.

Applicable scenarios:

- Temporary environments
- Simple distinction between development lines

---

### 4) Branch Name + Commit Sha

Example:

    manual-web:dev-a1b2c3d

Characteristics:

- Allows identification of specific code commits.
- Enhances traceability significantly.
- Makes rollbacks clearer.

Pro### Step 3: Push the production-oriented understanding tag

Execute:

    docker tag manual-web:v10 your-Harbor-domain/test/manual-web:dev-e8f7a6b-1101
    docker push your-Harbor-domain/test/manual-web:dev-e8f7a6b-1101

### Step 4: Check on the Harbor page

Go to:

- Project: `test`
- Repository: `manual-web`

Verify that at least these tags appear in the repository:

- `latest`
- `v10`
- `dev-e8f7a6b-1101`

### Understanding to be Established at This Step

From Harbor's perspective, it is normal to maintain multiple tags under the same repository.  
The key points are:

- Which tag is used for learning and observation
- Which tag is used for production deployment
- Which tag is suitable for tracking and rollback purposes

---

## Section 10: Practical Operation – Updating Deployments with Different Tags

This section involves an important comparison.

### Option A: Using a Learning Tag for Deployment

Execute:

    kubectl -n test set image deployment/manual-web manual-web=your-Harbor-domain/test/manual-web:v10

Check the rollout status:

    kubectl -n test rollout status deployment/manual-web

Verify within the cluster:

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

Execute inside the container:

    wget -qO- http://manual-web.test.svc.cluster.local

### Option B: Using a Production-oriented Understanding Tag for Deployment

Execute:

    kubectl -n test set image deployment/manual-web manual-web=your-Harbor-domain/test/manual-web:dev-e8f7a6b-1101

Check the rollout status:

    kubectl -n test rollout status deploymentmanual-web

Verify the page content again.

### Understanding to be Established at This Step

From the results, the two tags may refer to the same content.  
However, from a "version tracking" perspective, they differ significantly:

- `v10` is easy for humans to understand version evolution
- `dev-e8f7a6b-1101` helps track the image's origin and build process

This is why the latter is more recommended in production environments.

---

## Section 11: Recommended Tag Rules at This Stage

It is suggested that you memorize these rules directly and follow them later on.

### Rule 1: Do Not Use `latest` as the Only Official Release Identifier

It can exist, but it is not advised to rely solely on it in the long term.

### Rule 2: During the Learning Phase, You Can Retain Simple Tags Like `v1/v2/v3/v10`

These tags are most helpful for observing experimental results.

### Rule 3: For Images That Are Really to Be Used by Harbor/Deployment/Helm Values, Try to Apply a Production-oriented Understanding Tag

The preferred format is:

    Branch Name-Commit SHA-Pipeline ID

For example:

    dev-e8f7a6b-1101

### Rule 4: Deployments and Helm Values Should Refer to Specific Tags Instead of `latest` in the Long Term

This ensures more definitive releases and makes troubleshooting and rollback clearer.

---

## Section 12: A Small Experiment Intentionally Creating Confusion with `latest`

This experiment is not meant to cause problems but to help build intuition.

### Step 1: Prepare a new content, such as `v11-latest-test`

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

### Step 2: Rebuild and Continue to Overwrite `latest`

    docker build -t manual-web:latest .
    docker tag manual-web:latest your-Harbor-domain/test/manual-web:latest
    docker push your-Harbor-domain/test/manual-web:latest

### Step 3: If Your Deployment Also Refers to `latest` in the Long Term

Then, even though you know that `latest` in Harbor has changed, you will be unclear about which version it actually is.

### Understanding to be Established at This Step

The issue is not that `latest` cannot be used, but that:

**It is not suitable as a long-term, traceable version identifier.**

You cannot determine from `latest` alone:

- Which code change
- Which build occurred
- Which release was made

---

##