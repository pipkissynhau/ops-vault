# 09-CI-CD ReleaseActual: From Building Images to Deployment to Kubernetes

## Document Explanation

This article is the 9th note in the 08-CI-CD learning path.

The previous 01 to 08 have separately studied:

1. A minimal manual release pipeline
2. Segment understanding of release pipelines
3. Image building and production-style Tag
4. Projects, repositories, tags, and Robot accounts in Harbor
5. Minimal pipeline understanding of GitLab CI
6. Minimal pipeline understanding of Jenkins Pipeline
7. Deployment, ReplicaSet, Pod, and rolling update mechanisms
8. Minimal templating and rollback of Helm

This article doesn't focus on a single point anymore, but connects the previous content into a complete practical main line:

Application content  
→ Build image  
→ Apply production-style Tag  
→ Push to Harbor  
→ Choose kubectl or Helm to deploy to K8s  
→ Observe rollout  
→ Validate results  
→ Troubleshoot by segmenting the pipeline when issues occur

This article continues to align with the current environment:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private repository
- `test` namespace as the experimental environment

## Tags

#Kubernetes #CI-CD #Harbor #Helm #Deployment #rollout #MirrorBuild #ReleaseTheField. #I'llTakeYourNotes.

## Learning Objectives

After completing this article, you should master:

1. Be able to independently complete the entire minimal release pipeline
2. Be able to switch between kubectl and Helm deployment methods
3. Be able to use more production-like image Tags
4. Be able to observe the rollout process and determine if the deployment was successful
5. Be able to locate which segment of the pipeline failed when deployment fails
6. Be able to explain the entire release pipeline

## This Article's Experiment Design

This article is divided into two optional deployment paths:

### Path A: kubectl Deployment

Suitable for understanding the most direct deployment action.

### Path B: Helm Deployment

Suitable for understanding templating, parameterization, and release management.

Both paths share the same image build result, so you'll better understand:

- Image building is the first half
- kubectl / Helm are just two different entry points for the second half

## Experiment Preparation

### 1) Enter the Experimental Directory

The application content directory continues to use the previously existing directory:

    cd ~/08-ci-cd/01-manual-release

The Helm directory continues to use the previously existing directory:

    cd ~/08-ci-cd/08-helm-lab/manual-web

### 2) Confirm Cluster and Namespace are Normal

Execute:

    kubectl get nodes
    kubectl get ns
    kubectl -n test get deploy,pods,svc

### 3) Confirm Harbor is Accessible

Execute:

    docker login your Harbor domain

### 4) Confirm Helm is Available

Execute:

    helm version

## Part 1: Prepare a New Release Version

This article recommends starting with a completely new version, such as `v9`.

### Step 1: Modify Application Content

Enter the application directory:

    cd ~/08-ci-cd/01-manual-release

Modify `index.html`:

    cat > index.html <<'EOF'
    <html>
      <head>
        <meta charset="utf-8">
        <title>manual release v9</title>
      </head>
      <body>
        <h1>Hello K8s CI/CD</h1>
        <p>version: v9</p>
      </body>
    </html>
    EOF

Check the content:

    cat index.html

### Current Understanding to Establish

The starting point of the release pipeline is always "content change".  
In this experiment, this content change is the page version changing from the old value to `v9`.

## Part 2: Build Image

### Step 1: First build a local learning tag

Execute:

    docker build -t manual-web:v9 .

Check the image:

    docker images | grep manual-web

### Step 2: Local Run Verification

Execute:

    docker run -d --name manual-web-v9 -p 18080:80 manual-web:v9
    curl http://127.0.0.1:18080
    docker rm -f manual-web-v9

### Expected Phenomenon

The output should include:

    version: v9

### Current Understanding to Establish

Here we are confirming:

- The new content has entered the image
- The image can run locally
- The problem hasn't expanded to Harbor and K8s yet

## Part 3: Apply More Production-Like Tags to the Image

Currently, we don't just keep the learning-type `v9`, but also add a group of production-style tags.

### Assume the Build Information is as Follows

- Branch name: `dev`
- Commit ID: `f9a8b7c`
- Pipeline ID: `901`

### Step 1: Apply Production-Style Tags to the Local Image

Execute:

    docker tag manual-web:v9 manual-web:dev-f9a8b7c-901

Check the image:

    docker images | grep manual-web

### Current Understanding to Establish

Now the same image has at least two perspectives:

- `v9`: Convenient for current learning identification
- `dev-f9a8b7c-901`: Closer to production tracking

## Part 4: Push the Image to Harbor

### Step 1: Apply the Full Address Tag to Harbor

Execute:

    docker tag manual-web:v9 your Harbor domain/test/manual-web:dev-f9a8b7c-901

Example:

    docker tag manual-web:v9 harbor.example.com/test/manual-web:dev-f9a8b7c-901

### Step 2: Push to Harbor

Execute:

    docker push your Harbor domain/test/manual-web:dev-f9a8b7c-901

### Step 3: Confirm on the Harbor Page

Confirm:

- Project: `test`
- Repository: `manual-web`
- Tag: `dev-f9a8b7c-901`

### Current Understanding to Establish

By this point, the image has truly entered the unified repository.  
Subsequently, whether using kubectl or Helm, the references should be to this real tag in Harbor.

## Part 5: Path A - Using kubectl for Deployment

This section is suitable for understanding the most direct and primitive deployment action.

### Step 1: Directly Update Deployment Image

Execute:

    kubectl -n test set image deployment/manual-web manual-web=your Harbor domain name/test/manual-web:dev-f9a8b7c-901

### Step 2: Observe the Update Process Simultaneously

Recommended to run three windows separately:

    kubectl -n test get deploy -w
    kubectl -n test get rs -w
    kubectl -n test get pods -w

### Step 3: Check Rollout Status

Execute:

    kubectl -n test rollout status deployment/manual-web

### Step 4: Enter the Cluster to Verify Page Content

Execute:

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

After entering, execute:

    wget -qO- http://manual-web.test.svc.cluster.local

### Expected Phenomenon

Should see:

    version: v9

### Understanding to Establish for Path A

The entire process here is:

- A new image exists in Harbor
- The Deployment's image is changed to the new tag
- K8s triggers a rolling update
- Service continues to provide access
- Business content becomes v9

This is the most direct way of deployment.

## Part 6: Path B - Using Helm for Deployment

This section is suitable for understanding templating and parameterized deployment.

### Step 1: Enter Helm Chart Directory

Execute:

    cd ~/08-ci-cd/08-helm-lab/manual-web

### Step 2: Modify values.yaml's image.tag

First check current configuration:

    cat values.yaml

Find:

    image:
      repository: your Harbor domain name/test/manual-web
      tag: "old value"

Change `tag` to:

    tag: "dev-f9a8b7c-901"

Save and verify again:

    cat values.yaml

### Step 3: First Check Rendering Results

Execute:

    helm template manual-web . -n test

Focus on confirming whether the rendered image has become:

    your Harbor domain name/test/manual-web:dev-f9a8b7c-901

### Step 4: Execute Helm Upgrade

Execute:

    helm upgrade manual-web . -n test

### Step 5: Check Release Status and History

Execute:

    helm list -n test
    helm history manual-web -n test

### Step 6: Observe Pod Update Process

Execute:

    kubectl -n test get rs -w
    kubectl -n test get pods -w
    kubectl -n test rollout status deployment/manual-web

### Step 7: Enter the Cluster to Verify Page Content

Execute:

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

After entering, execute:

    wget -qO- http://manual-web.test.svc.cluster.local

### Expected Phenomenon

Should see:

    version: v9

### Understanding to Establish for Path B

The difference from kubectl is:

- You are not directly modifying the Deployment
- Instead, you first modify values
- Helm re-renders the template
- Then updates the Release
- The Release drives K8s resource changes

This is Helm's role in the current mainline.

## Part 7: Comparing kubectl Deployment and Helm Deployment

This section is very important. After completing both paths, you must compare them yourself.

### kubectl Deployment

Features:

- Direct
- Fewer actions
- Suitable for beginners and small resources
- You clearly know what you changed

Limitations:

- Maintenance cost increases with more resources
- Not suitable for scenarios with many scattered parameters
- Rollback mainly relies on Deployment's own history

### Helm Deployment

Features:

- Manage parameters through values
- Templating is more suitable for complex applications
- Has release history
- More suitable for integration with GitLab CI/Jenkins later

Limitations:

- An extra layer of template rendering
- Stronger abstraction initially

### Understanding to Establish at This Stage

Both are not separate from image building and Harbor.  
They are simply "two different paths to send the image into K8s after it's ready".

## Part 8: Troubleshooting When Deployment Fails

This section is one of the most important practical skills in this article.

In the future, when your deployment fails, don't immediately say "CI/CD is problematic". Instead, troubleshoot by segmenting the chain of events.

### Segment 1: Content Phase

Check:

- Whether `index.html` has actually changed
- Whether the content you're deploying is indeed the current version

Command:

    cat index.html

### Segment 2: Image Build Phase

Check:

- Whether `docker build` succeeded
- Whether the local content is correct

Command:

    docker images | grep manual-web
    docker run -d --name test-v9 -p 18080:80 manual-web:v9
    curl http://127.0.0.1:18080
    docker rm -f test-v9

### Segment 3: Image Push Phase

Check:

- Whether Harbor actually has this tag

Command:

    docker push ...
    Manually confirm the tag on the Harbor page

### Segment 4: Cluster Configuration/Deployment Phase

If using kubectl:

- Whether the Deployment's image has actually been updated

Command:

    kubectl -n test describe deploy manual-web | grep -A3 Image

If using Helm:

- Whether values have actually changed
- Whether the rendered result is correct
- Whether the release was successfully upgraded

Command:

    cat values.yaml
    helm template manual-web . -n test
    helm history manual-web -n test

### Segment 5: Cluster Runtime Phase

Check:

- Is the Pod running  
- Are `ErrImagePull`, `ImagePullBackOff`, and `CrashLoopBackOff` appearing  

Commands:  

    kubectl -n test get pods  
    kubectl -n test describe pod Pod name  
    kubectl -n test logs Pod name  

### Section 6: Release Verification Phase  

Checks:  

- Was the rollout successful  
- Has the page content actually updated  

Commands:  

    kubectl -n test rollout status deployment/manual-web  
    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh  
    wget -qO- http://manual-web.test.svc.cluster.local  

## Section 9: Intentionally Creating a Failure Exercise  

This section is strongly recommended to complete, as it quickly builds troubleshooting intuition.  

### Exercise Objective  

Deliberately change the image tag in the Deployment or Helm values to a tag that does not exist in Harbor, for example:  

    not-exist-999  

### Path A: Creating an Error via kubectl  

Execute:  

    kubectl -n test set image deployment/manual-web manual-web=your Harbor domain/test/manual-web:not-exist-999  

### Path B: Creating an Error via Helm  

In `values.yaml`, change:  

    tag: "dev-f9a8b7c-901"  

to:  

    tag: "not-exist-999"  

Then execute:  

    helm upgrade manual-web . -n test  

### Observation Points  

Execute:  

    kubectl -n test get pods  
    kubectl -n test describe pod Pod name  
    kubectl -n test rollout status deployment/manual-web  

### What You Should See  

Typically, you will see:  

- `ErrImagePull`  
- `ImagePullBackOff`  
- rollout is stuck  

### Significance of This Exercise  

This exercise builds a critical response:  

**Once a Pod cannot pull an image, prioritize checking whether the tag exists in Harbor.**  

## Section 10: Reconnecting Steps 01 to 08 into a Complete Mainline  

Now you should be able to explain the entire mainline:  

1. Start with application content changes  
2. Build a local image via Dockerfile  
3. Use a production-style tag to identify the image  
4. Push to Harbor to form a unified image version  
5. Use kubectl or Helm to deploy this image version into K8s  
6. Deployment begins rolling updates  
7. Finally verify the release result via rollout and actual access  
8. When issues occur, troubleshoot segment by segment along the chain  

If you can smoothly explain these 8 sentences, steps 01 to 08 are truly connected.  

## Section 11: This Article's Practice Exercises  

### Exercise 1: Independently Complete a v10 Release  

Requirements:  

- Modify `index.html` yourself  
- Build the image  
- Apply a production-style tag  
- Push to Harbor  
- Choose either kubectl or Helm for deployment  
- Verify the page results  

### Exercise 2: Walk the Same Version Through Both kubectl and Helm Paths  

Requirements:  

- First use a new tag with kubectl  
- Then use another new tag with Helm  
- Compare the different sensations between the two deployment methods  

### Exercise 3: Answer the Following 5 Questions Yourself  

1. What are the minimum stages a complete release must go through  
2. Where does Harbor fall in the entire chain  
3. What are the main differences between kubectl and Helm deployments  
4. Why does a successful rollout still not guarantee the release is absolutely flawless  
5. Why should you troubleshoot segment by segment when a release fails  

If you can clearly explain these 5 questions, you've mastered this article.  

## Content You Should Be Able to Explain After This Article  

After completing this article, you should be able to explain the following:  

A complete CI/CD release practice essentially starts with application content changes, builds an image first, then applies a traceable production-style tag, pushes to Harbor to form a unified image version.  
Then you can directly update the Deployment via kubectl, or modify values and upgrade the release via Helm.  
Regardless of the method, it will ultimately trigger Kubernetes' rolling update mechanism, and the release success is judged by the rollout status and business access results.  
If the release fails, you should troubleshoot segment by segment along the content, build, repository, configuration, deployment, and verification stages, rather than vaguely saying the pipeline failed.  

## Common Issues and Troubleshooting Directions  

### Issue 1: I built successfully, but K8s still uses the old version  

Prioritize checking:  

- Whether you actually pushed the new tag  
- Whether the Deployment/values truly reference the new tag  
- Whether the tag truly exists in Harbor  

### Issue 2: Helm upgrade succeeded, but the page hasn't changed  

Prioritize checking:  

- Whether the tag in `values.yaml` has truly changed  
- Whether the image rendered by `helm template` is correct  
- Whether the Pod has truly completed the update  

### Issue 3: Which is better between kubectl and Helm  

Currently, it's not a choice between the two.  
You should go through both paths, first establishing a clear contrast.  

## Key Takeaways from This Article  

After completing this article, you should master:  

1. How to independently walk through the minimum release chain  
2. How to use more production-like image tags  
3. How to deploy using both kubectl and Helm  
4. How to observe rollout results  
5. How to troubleshoot release failures by segment  

## One-Sentence Summary  

The core of CI/CD release practice is not a single tool, but to stably walk through the fixed chain of "content changes → build image → push to Harbor → deploy to K8s → verify results," and be able to precisely locate issues along the chain when a release fails.  

## Next Article  

The next article will enter:  

10-Release Verification, Rollback, and Common Issue Troubleshooting: From Successful Deployment to Stable Delivery  

The next article will focus on:  

- How to properly verify a successful release  
- How to use Deployment rollback and Helm rollback  
- How to quickly segment locate common failure phenomena  
- How to elevate "being able to deploy" to "being able to reliably deploy, rollback, and troubleshoot"