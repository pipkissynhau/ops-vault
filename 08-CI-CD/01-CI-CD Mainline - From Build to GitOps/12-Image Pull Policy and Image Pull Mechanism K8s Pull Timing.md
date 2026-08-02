# 12-imagePullPolicy and Image Pull Mechanism: When Does K8s Pull New Images

## Documentation Notes

This is the 12th note in the 08-CI-CD learning path.

The previous article has clarified the image tag strategy, and this article continues the discussion to solve a very common and often confusing issue in deployments:

**Why might K8s not immediately get the "new image" you think you pushed?**

This issue is typically related to the following:

- Has the image in Deployment changed?
- What is imagePullPolicy?
- Does the node have a cached image locally?
- Are you repeatedly overwriting with the same tag?

This article continues to align with the current environment:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private registry
- `test` namespace

## Tags

#Kubernetes #CI-CD #imagePullPolicy #MirrorPull #Harbor #containerd #latest #DisseminationMechanism #I'llTakeYourNotes.

## Learning Objectives

After completing this article, you should understand:

1. When K8s does not always re-pull images
2. The differences between the three common values of `imagePullPolicy`
3. Why "repeatedly overwriting with the same tag" easily creates pitfalls
4. Why explicitly using a new tag is more stable than repeatedly overwriting an old tag
5. Design and verify an image pull experiment in the current environment
6. Explain why the result differs when a Deployment changes or does not change the image

## This Article's Experiment Outline

This article is divided into 4 sections:

1. First understand what K8s looks at when pulling images
2. Understand `IfNotPresent`, `Always`, and `Never` separately
3. Conduct a comparative experiment using current Harbor + Deployment environment
4. Summarize the recommended release strategy for this stage

---

## Part 1: Establish a Core Understanding - K8s Does Not Always Re-Pull Images Unconditionally

Many people have a misconception when doing their first deployment:

- The image has been pushed to Harbor
- The Pod is restarted
- The node should immediately get the latest image

This understanding is incomplete.

When creating a Pod, K8s decides whether to re-pull an image based on the following factors:

1. Image address
2. Image tag
3. `imagePullPolicy`
4. Whether the node already has this image locally
5. Container runtime processing logic

So "push success" does not automatically mean "the node will definitely get the new image".

### Understanding to Establish at This Stage

When encountering image-related issues in the future, do not only think:

- Does Harbor have the image?

You should also consider:

- Does K8s have a reason to re-pull this image?

---

## Part 2: First Check the Current Deployment's image and imagePullPolicy

### Step 1: Check Current Deployment

Run:

    kubectl -n test get deploy manual-web -o yaml | grep -A10 "image:"

### Key Observations

You need to check two things:

1. What is the current image address and tag in `image`
2. Is `imagePullPolicy` explicitly written?

### Understanding to Establish at This Stage

If the Deployment does not explicitly write `imagePullPolicy`, K8s will handle it based on default rules.  
But in this stage, it is more recommended to:

**Do not rely on default assumptions, but explicitly write them out.**

---

## Part 3: Three Common imagePullPolicy Values

### 1) IfNotPresent

Meaning:

- If the node does not have this image, pull it from the registry
- If the node already has this image, prioritize using the local image

Advantages:

- Faster startup
- Reduced redundant pulls
- Commonly used for explicit version tags

Risks:

- If you push the same tag repeatedly, the node may continue using the old cache

Recommended Use:

- Explicit version tag scenarios
- Normal production release mainline

---

### 2) Always

Meaning:

- Attempt to pull the image from the registry every time a container is created

Advantages:

- More likely to get the current image content from the registry
- More "safe" for tags like `latest`

Risks:

- More dependent on registry availability each time
- Cannot replace proper tag strategies
- If the registry is slow or the network is poor, startup will be more affected

Recommended Use:

- Temporary verification
- For experimental scenarios with `latest` or frequently updated tags

---

### 3) Never

Meaning:

- Never pull the image from the registry, only use the local image on the node

Advantages:

- Can be used in some special offline environments

Risks:

- Basically unsuitable for normal release scenarios
- The node fails if the image is not present

Recommended Use:

- A few special tests
- Not suitable for the mainline

---

## Part 4: Understanding Default Behavior

At this stage, remember a practical judgment without getting into theBottom source code.

### Common Default Assumptions

- If the tag is `latest`, the default behavior usually leans toward `Always`
- If the tag is not `latest`, the default behavior usually leans toward `IfNotPresent`

### But the more recommended approach at this stage is

**Explicitly write `imagePullPolicy` in YAML / Helm values templates.**

This way, you don't have to guess when you see the configuration.

---

## Part 5: Why Repeatedly Overwriting the Same Tag is Dangerous

This section must be thoroughly explained because it's one of the most common pitfalls in releases.

Assume the current Deployment is written as:

    image: harbor.example.com/test/manual-web:v12

Later you:

1. Modify the page content
2. Rebuild
3. Still push to `v12`

You might easily assume:

- The `v12` in Harbor is already the new content
- K8s will get the new content as long as the Pod is restarted

But this is not necessarily true.

Because if:

- `imagePullPolicy` is `IfNotPresent`
- The node already has `v12`
- The Deployment's image field hasn't changed

The node may continue using the existing image cache.

### Understanding to Establish at This Stage

Repeatedly overwriting the same tag breaks the "determinism" of releases.

You'll start to be unsure:

- What content this tag corresponds to now
- Whether the node is getting the old or new image
- Whether the rollout triggered actually changed to the new version

Therefore:

**The most stable way in the release mainline is not to repeatedly overwrite old tags, but to use a clear new tag for each release.**

---

## Part 6: Why Using a Clear New Tag is More Stable

If you use a new tag for each release, for example:

- `v12`
- `v13`
- `dev-a1b2c3d-1201`
- `dev-b2c3d4e-1202`

So every time you release, the image address referenced by Deployment itself changes.

This means:

- K8s finds it easier to determine this is a new release target
- You find it easier to track versions yourself
- Rollbacks become clearer
- The risk of node caching misleading you is smaller

### Understanding to establish at this step

`imagePullPolicy` is important, but a more fundamental and stable strategy is:

**Give the image a clear new tag every time you release.**

---

## Part 7: Practical Operation 1 - Do a stable release with clear new tag + IfNotPresent

This section first implements the recommended path.

### Step 1: Prepare new version content

Enter the application directory:

    cd ~/08-ci-cd/01-manual-release

Modify `index.html`:

    cat > index.html <<'EOF'
    <html>
      <head>
        <meta charset="utf-8">
        <title>manual release v12</title>
      </head>
      <body>
        <h1>Hello K8s CI/CD</h1>
        <p>version: v12</p>
      </body>
    </html>
    EOF

### Step 2: Build and push with a clear new tag

Execute:

    docker build -t manual-web:v12 .
    docker tag manual-web:v12 Yours.HarborDomain Name/test/manual-web:v12
    docker push Yours.HarborDomain Name/test/manual-web:v12

### Step 3: Confirm v12 exists in Harbor

Go to the Harbor page to confirm:

- Repository: `manual-web`
- Tag: `v12`

### Step 4: Explicitly set imagePullPolicy to IfNotPresent in Deployment

If you're currently using native YAML, edit the Deployment section to ensure the container configuration is similar to this:

    containers:
      - name: manual-web
        image: Yours.HarborDomain Name/test/manual-web:v12
        imagePullPolicy: IfNotPresent

If you're using Helm, explicitly write the corresponding fields in values or templates and upgrade.

### Step 5: Deploy to K8s

If you're currently using a direct Deployment:

    kubectl -n test set image deployment/manual-web manual-web=Yours.HarborDomain Name/test/manual-web:v12

Then check the status:

    kubectl -n test rollout status deployment/manual-web

### Step 6: Verify page content

Execute:

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

After entering, execute:

    wget -qO- http://manual-web.test.svc.cluster.local

### Expected Phenomenon

You should see:

    version: v12

### Understanding to establish at this step

This path is the most recommended at this stage:

- Clear new tag
- Explicitly write `IfNotPresent`

This makes the release behavior most controllable and stable.

---

## Part 8: Practical Operation 2 - Experience why same tag overwrite is unstable

This section intentionally takes an "unrecommended path" to let you personally feel the differences.

### Step 1: Modify page content to v12-hotfix

    cat > index.html <<'EOF'
    <html>
      <head>
        <meta charset="utf-8">
        <title>manual release v12-hotfix</title>
      </head>
      <body>
        <h1>Hello K8s CI/CD</h1>
        <p>version: v12-hotfix</p>
      </body>
    </html>
    EOF

### Step 2: Rebuild but continue using the same v12 tag

    docker build -t manual-web:v12 .
    docker tag manual-web:v12 Yours.HarborDomain Name/test/manual-web:v12
    docker push Yours.HarborDomain Name/test/manual-web:v12

### Step 3: Don't change Deployment image, only restart Pod or re-roll

For example, you can try:

    kubectl -n test rollout restart deployment/manual-web

Then check:

    kubectl -n test rollout status deployment/manual-web

### Step 4: Verify page content again

Execute:

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

After entering, execute:

    wget -qO- http://manual-web.test.svc.cluster.local

### Possible Phenomena

You'll find the result of this step isn't as reassuring as with a clear new tag.  
Even if the final content changes, you'll discover the issues with this path:

- You find it hard to immediately determine which real content the Deployment corresponds to
- You blur the concept of "release version"
- Troubleshooting and rollback become more uncertain

### Understanding to establish at this step

The problem isn't just "whether the new image was pulled or not". The bigger issue is:

**Same tag overwrite makes the version identity itself distorted.**

---

## Part 9: Practical Operation 3 - Feel policy differences with Always

This section isn't recommended as the main path, but serves only for understanding contrast.

### Step 1: Change imagePullPolicy in Deployment to Always

Set in native YAML or templates:

    imagePullPolicy: Always

### Step 2: Keep image still using some tag, e.g. latest or same tag

For example:

    image: Yours.HarborDomain Name/test/manual-web:latest

### Step 3: Re-push a new latest

Prepare new content: /think

```bash
cat > index.html <<'EOF'
<html>
  <head>
    <meta charset="utf-8">
    <title>manual release latest-v12</title>
  </head>
  <body>
    <h1>Hello K8s CI/CD</h1>
    <p>version: latest-v12</p>
  </body>
</html>
EOF
```

Build and push:

```bash
docker build -t manual-web:latest .
docker tag manual-web:latest your Harbor domain/test/manual-web:latest
docker push your Harbor domain/test/manual-web:latest
```

### Step 4: Restart Deployment

```bash
kubectl -n test rollout restart deployment/manual-web
kubectl -n test rollout status deployment/manual-web
```

### Step 5: Verify Page Content

Execute cluster internal access verification.

### Current Understanding to Establish

`Always` indeed makes it easier for nodes to get the content corresponding to the "current tag" in the repository, but it doesn't solve the root problem:

- Your tag is still not traceable enough
- Your rollback is still not clear enough

Therefore:

**Always cannot replace a reasonable image tag strategy.**

---

## Part 10: Current Recommended Release Combination

Combined with your current environment and learningMain, the most recommended is:

### Recommended Combination

- Use a clear new tag every release
- Explicitly write `imagePullPolicy: IfNotPresent` in Deployment / Helm

### Reasons

1. The tag itself clearly indicates the version
2. Changes in the image field will clearly trigger release updates
3. The behavior of whether nodes re-pull is easier to predict
4. Rollback is simpler
5. Version clarity is better on Harbor's page

### Current Understanding to Establish

This combination is more stable than "long-term latest + Always", and is more suitable for later integration with GitLab CI / Jenkins / Helm.

---

## Part 11: How to Connect This Article with Article 11

You have already learned in the previous article:

- latest
- v1 / v2 / v3
- Branch name + commit sha + pipeline id

This article continues to tell you:

- Tags are not just about names
- It affects Kubernetes image pull behavior and release determinism

Therefore, these two articles should be understood together:

### Article 11 Solves

"How should images be named"

### Article 12 Solves

"What does Kubernetes do when pulling images after naming"

Together, these two articles make the image release section more complete.

---

## Part 12: This Article's Practice Exercise

### Exercise 1: Perform a Clear Tag Release

Requirements:

- Build a `v13`
- Push to Harbor
- Deployment references `v13`
- Set `IfNotPresent`
- Verify page content

### Exercise 2: Perform a Same Tag Overwrite Experiment

Requirements:

- Change the content of `v13`
- Continue pushing to `v13`
- Don't change the Deployment's tag, only try restart
- Observe why this method is unstable

### Exercise 3: Answer the Following 4 Questions Yourself

1. What is the core difference between `IfNotPresent` and `Always`
2. Why is same tag re-push dangerous
3. Why is clear new tag more stable
4. Why can't `Always` replace a reasonable tag strategy

If you can explain these 4 questions yourself, you've mastered this article.

---

## Content to Be Able to Explain After This Article

After completing this article, it's recommended to be able to explain the following:

Kubernetes doesn't decide whether to pull new images based solely on whether there's new content in Harbor, but rather by combining the image address, tag, imagePullPolicy, and node-local cache.  
`IfNotPresent` indicates that nodes will prioritize using local cache if available, while `Always` indicates that Kubernetes will attempt to pull images from the repository every time a container is created.  
The truly stable release method is not repeatedly overwriting the same tag, but using a clear new tag every release, such as `v12` or `dev-a1b2c3d-1201`.  
Combined with `IfNotPresent`, this makes image versions clearer, releases more stable, and rollbacks easier.

## Common Issues and Troubleshooting Directions

### Issue 1: I pushed a new image, but the Pod still shows old content

Check priority:

- Whether the Deployment's image has really changed to the new tag
- Whether this tag actually exists in Harbor
- What is the imagePullPolicy
- Whether you've overwritten the old tag

### Issue 2: I used Always, is it foolproof?

No.  
Always only affects pull behavior and cannot replace clear version management.

### Issue 3: Should I completely prohibit latest?

Currently, don't take extreme measures.  
You can keep latest for local or temporary experiments, but avoid relying solely on latest in formal release pipelines.

---

## Key Points to Master This Article

1. When does Kubernetes pull new images
2. The difference between `IfNotPresent`, `Always`, and `Never`
3. Why same tag overwrite is risky
4. Why clear tags are more stable
5. The recommended release combination at this stage

## One-Sentence Summary

The stability of a release depends not only on whether there's new image in Harbor, but also on whether the image tag is clear, whether the Deployment's image has truly changed, and whether the imagePullPolicy matches your release strategy.