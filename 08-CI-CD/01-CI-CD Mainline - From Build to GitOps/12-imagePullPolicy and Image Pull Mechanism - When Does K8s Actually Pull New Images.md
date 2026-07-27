# 12-imagePullPolicy and Image Pull Mechanism: When Does K8s Actually Pull New Images?

## Documentation Note

This article is the 12th note in the 08-CI-CD learning path.

The previous article explained the image tag policy in detail. In this one, we will continue to address a very common and confusing issue in deployment:

**Why doesn’t K8s immediately get the “new image” you expect even though the image has been re-pushed?**

This problem is usually related to the following factors:

- Has the image in the Deployment changed?
- What is the `imagePullPolicy`?
- Does the node have a cached image locally?
- Are you repeatedly overwriting the same tag?

This article continues to use the current environment:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private repository
- `test` namespace

## Tags

#Kubernetes #CI-CD #imagePullPolicy #Image Pull #Harbor #containerd #latest #Deployment Mechanism #Practical Notes

## Learning Objectives

After completing this article, you should be able to:

1. Understand that K8s does not always pull new images automatically.
2. Differentiate between the three common values of `imagePullPolicy`.
3. Recognize why repeatedly overwriting the same tag can cause problems.
4. Understand why using a clear new tag is more reliable than overwriting an old one.
5. Design and verify an image pull experiment in the current environment.
6. Clearly explain why changing or not changing the `image` in a Deployment can result in different outcomes.

## Main Experimental Approach of This Article

This article is divided into 4 sections:

1. First, understand what K8s considers when pulling images.
2. Then, separately understand the meanings of `IfNotPresent`, `Always`, and `Never`.
3. Conduct comparative experiments using the current Harbor + Deployment environment.
4. Summarize the recommended deployment strategy for this stage.

---

## Part 1: Establish a Core Understanding – K8s Does Not Always Pull New Images Unconditionally

Many people have an intuition when they first start deploying:

- Once the image is pushed to Harbor,
- The Pod will be rebuilt,
- And the node should get the latest image.

This understanding is incomplete.

When creating a Pod, K8s considers the following factors to decide whether to pull a new image:

1. Image address
2. Image tag
3. `imagePullPolicy`
4. Whether the node already has this image locally
5. The processing logic during container runtime

Therefore, “successful push” does not automatically mean that “the node will definitely get the new image.”

### Understanding to Establish at This Step

In the future, when encountering image-related issues, don’t just think about:

- Whether there is an image in Harbor.

You also need to consider:

- Whether K8s has a reason to pull this image again.

---

## Part 2: First, Check the Current Deployment’s `image` and `imagePullPolicy`

### Step 1: View the Current Deployment

Execute:

    kubectl -n test get deploy manual-web -o yaml | grep -A10 "image:"

### Key Observations

You need to check two things:

1. What image address and tag are currently specified for `image`.
2. Whether `imagePullPolicy` is explicitly defined.

### Understanding to Establish at This Step

If the Deployment does not specify `imagePullPolicy`, K8s will use default rules.  
However, it is currently recommended that you:

**Do not rely on default assumptions; instead, explicitly define it whenever possible.**

---

## Part 3: The Three Common `imagePullPolicy` Values

### 1) IfNotPresent

Meaning:

- If the node does not have this image locally, pull it from the repository.
- If the node already has this image locally, use the local copy instead.

Advantages:

- Faster startup
- Reduces repeated pulls
- Useful for specifying a clear version tag

Risks:

- If you re-push an image with the same tag, the node may continue to use the old cache.

Suitable for:

- Scenarios where a specific version tag is required
- Main production deployment pipelines

---

### 2) Always

Meaning:

- Try to pull the image from the repository every time a container is created.

Advantages:

- More likely to get the latest image content from the repository
- Relatively “safer” for tags like `latest`

Risks:

- Depends more on the availability of the repository during startup
- Cannot replace a proper tag strategy
- Slow startup may occur if the repository is slow or the network is poor

Suitable for:

- Temporary verification purposes
- Experiments with `latest` or frequently updated tags

---

### 3) Never

Meaning## Section Eight: Practical Exercise Two – Understanding the Instability of Same Tag Overwrites

This section intentionally presents a “non-recommended approach” to help you personally experience the differences.

### Step One: Modify the page content to v12-hotfix

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

### Step Two: Rebuild but continue using the same v12 tag

    docker build -t manual-web:v12 .
    docker tag manual-web:v12 your-Harbor-domain/test/manual-web:v12
    docker push your-Harbor-domain/test/manual-web:v12

### Step Three: Do not change the Deployment image; only restart the Pod or perform a rollout

For example, you can try:

    kubectl -n test rollout restart deployment/manual-web

Then check:

    kubectl -n test rollout status deploymentmanual-web

### Step Four: Verify the page content again

Execute:

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

Inside the script, execute:

    wget -qO- http://manual-web.test.svc.cluster.local

### Possible Observations

You will find that the result of this step is not as reassuring as using a “new tag.” Even if the content ultimately changes, you will notice that this approach has the following issues:

- It is difficult to quickly determine which version of the content corresponds to the Deployment.
- The concept of “release version” becomes blurred.
- Troubleshooting and rolling back become more uncertain.

### Key Understanding for This Step

The problem is not just whether a new image is pulled; the larger issue is that using the same tag overwrites distorts the identity of the version itself.

---

## Section Nine: Practical Exercise Three – Experiencing Policy Differences with Always

This section is not recommended as the main approach but serves as a comparative understanding.

### Step One: Change the imagePullPolicy in the Deployment to Always

Set it in the native YAML or template:

    imagePullPolicy: Always

### Step Two: Continue using the same tag for the image, such as latest or the same tag

For example:

    image: your-Harbor-domain/test/manual-web:latest

### Step Three: Push a new latest version

Prepare new content:

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

Build and push:

    docker build -t manual-web:latest .
    docker tag manual-web:latest your-Harbor-domain/test/manual-web:latest
    docker push your-Harbor-domain/test/manual-web:latest

### Step Four: Restart the Deployment

    kubectl -n test rollout restart deploymentmanual-web
    kubectl -n test rollout status deploymentmanual-web

### Step Five: Verify the page content

Perform an in-cluster access check to verify.

### Key Understanding for This Step

`Always` does make it easier for nodes to retrieve the current version of content from the repository, but it does not address the fundamental issues:

- Your tag is still not traceable enough.
- Rolling back remains unclear.

Therefore:

`Always` cannot replace a proper image tag strategy.

---

## Section Ten: The Most Recommended Release Combination at This Stage

Based on your current environment and learning focus, the most recommended combination is:

### Recommended Combination

- Use a clear new tag for each release.
- Explicitly set `imagePullPolicy: IfNotPresent` in Deployment/Helm.

### Reasons

1. The tag itself clearly indicates the version.
2. Changes to the image field will trigger a clear release update.
3. It is easier to predict whether nodes will re-pull images.
4. Rolling back becomes simpler.
5. The version displayed on the Harbor page is clearer.

### Key Understanding for This Step

This combination is more stable than using “latest indefinitely” with `Always` and is better suited for integration with GitLab CI/Jenkins/Helm.

---

## Section Eleven: How to Connect These Two Articles

In the previous article, you learned about:

- latest
- v1 / v2 / v3
- branch name + commit sha + pipeline id

This article further explains that:

- Tags are more than just names; they affect how K8s pulls images