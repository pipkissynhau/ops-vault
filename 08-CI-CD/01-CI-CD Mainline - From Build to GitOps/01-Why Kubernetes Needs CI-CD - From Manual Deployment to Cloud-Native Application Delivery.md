# 01-Why K8s Needs CI-CD: From Manual Deployment to Cloud-Native Application Delivery

## Document Notes

This is the first note in the 08-CI-CD learning path.

This article, based on the existing experimental environment, uses a minimal deployable release experiment to establish an intuitive understanding of the Kubernetes application delivery pipeline. The focus is not on tool definitions, but on completing a manual deployment process first, then understanding why CI/CD will be introduced later based on the experimental process.

The default environment is as follows:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private registry
- `test` namespace as the experimental environment

## Tags

#Kubernetes #CI-CD #Harbor #Deployment #Service #ScrollUpdate #MirrorRelease #I'llTakeYourNotes.

## Learning Objectives

Complete a minimal manual deployment, and then complete a version update.

After completion, you should master the following:

1. Understand that Kubernetes ultimately runs images, not local source code files
2. Understand where Harbor is positioned in the delivery pipeline
3. Understand what steps a manual deployment typically includes
4. Understand why a Deployment triggers rolling updates after updating the image
5. Understand why these repetitive actions will later enter the CI/CD process

## The Delivery Pipeline to Complete in This Experiment

Page file  
→ Build image  
→ Push to Harbor  
→ Deploy to K8s  
→ Cluster internal access verification  
→ Modify content and re-release  
→ Observe rolling update process

## Experimental Preparation

### 1) Confirm Cluster Status

Execute:

    kubectl get nodes
    kubectl get ns

Confirm that the `test` namespace exists. If it doesn't exist, create it:

    kubectl create namespace test

### 2) Confirm Harbor is Accessible and Loggable

Execute:

    docker login your Harbor domain

Example:

    docker login harbor.example.com

### 3) Create Experimental Directory

Execute:

    mkdir -p ~/08-ci-cd/01-manual-release
    cd ~/08-ci-cd/01-manual-release

All subsequent files will be placed here.

## Step 1: Prepare v1 Page File

Create `index.html`:

    cat > index.html <<'EOF'
    <html>
      <head>
        <meta charset="utf-8">
        <title>manual release v1</title>
      </head>
      <body>
        <h1>Hello K8s CI/CD</h1>
        <p>version: v1</p>
      </body>
    </html>
    EOF

### Step Understanding

The `index.html` here represents "application content".  
Subsequent version changes essentially mean this content changes, then goes through image rebuilding and re-release.

## Step 2: Write Dockerfile

Create `Dockerfile`:

    cat > Dockerfile <<'EOF'
    FROM nginx:1.27
    COPY index.html /usr/share/nginx/html/index.html
    EXPOSE 80
    EOF

### Step Understanding

This step is about "image packaging".

Meaning:

- Base image uses nginx
- Copy local page file into the image
- The container starts with nginx providing static pages

We won't introduce Java, Maven, Jenkins, etc. here, first run the minimal delivery pipeline.

## Step 3: Build Local Image

Execute:

    docker build -t manual-web:v1 .

Check the image:

    docker images | grep manual-web

### Expected Phenomenon

You should see something like:

    manual-web    v1

### Step Understanding

Here you get a **local image**.  
It only exists on the current build machine and isn't yet a unified delivery item for the cluster.

## Step 4: Local Verify Image Content

Execute:

    docker run -d --name manual-web-v1 -p 18080:80 manual-web:v1

Check the page:

    curl http://127.0.0.1:18080

Expected output should include:

    Hello K8s CI/CD
    version: v1

After verification, delete the container:

    docker rm -f manual-web-v1

### Step Understanding

This step is image self-check.

First confirm the image itself is fine, then proceed to Harbor and K8s, making subsequent troubleshooting scope clearer.

## Step 5: Tag Image for Harbor

Change the image to the full Harbor address format.

Execute:

    docker tag manual-web:v1 your Harbor domain/test/manual-web:v1

Example:

    docker tag manual-web:v1 harbor.example.com/test/manual-web:v1

### Step Understanding

This step isn't "rebuilding the image", but adding a remote registry reference name to the local image, making it easier to push later.

## Step 6: Push Image to Harbor

Execute:

    docker push your Harbor domain/test/manual-web:v1

Example:

    docker push harbor.example.com/test/manual-web:v1

### Expected Phenomenon

After successful push, the Harbor page should show:

- Project: `test`
- Repository: `manual-web`
- Tag: `v1`

### Step Understanding

At this point, the image has truly become a "deliverable that can be uniformly pulled by the cluster".

## Step 7: Write Deployment and Service

Create `manual-web.yaml`:

cat > manual-web.yaml <<'EOF'
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: manual-web
      namespace: test
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: manual-web
      template:
        metadata:
          labels:
            app: manual-web
        spec:
          containers:
            - name: manual-web
              image: `你的Harbor域名`/test/manual-web:v1
              ports:
                - containerPort: 80
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: manual-web
      namespace: test
    spec:
      selector:
        app: manual-web
      ports:
        - protocol: TCP
          port: 80
          targetPort: 80
      type: ClusterIP
    EOF

### Understanding This Step

Two core objects are created here:

- Deployment: Manages application replicas and subsequent rolling updates
- Service: Provides a unified entry point for cluster access

## Step 8: Deploy to K8s

Execute:

    kubectl apply -f manual-web.yaml

Check resource status:

    kubectl -n test get deploy
    kubectl -n test get pods -o wide
    kubectl -n test get svc

### Expected Outcomes

Normally, you'll see:

- Deployment has been created
- Pod count reaches 2/2 gradually
- Service has been created

## Step 9: Check Pod Status

Check Pod status:

    kubectl -n test get pods -o wide

If abnormal, check details:

    kubectl -n test describe pod PodName

If Pod runs normally, check logs:

    kubectl -n test logs PodName

### Key Observations in This Step

If you see:

- `ErrImagePull`
- `ImagePullBackOff`

Prioritize checking:

1. Harbor address is correctly written
2. Project, repository, and tag exist genuinely
3. Nodes have trusted Harbor certificate
4. Private repository authentication is configured

Based on your existing Harbor/containerd configuration experience, this step is critical.

## Step 10: Access Service Internally in Cluster

Execute:

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

After entering, execute:

    wget -qO- http://manual-web.test.svc.cluster.local

### Expected Outcomes

Output should include:

    Hello K8s CI/CD
    version: v1

### Understanding This Step

This step verifies:

- Whether the Pod actually provides content
- Whether the Service correctly selects the Pod
- Whether cluster DNS is functioning normally
- Whether the current version is indeed v1

The first manual deployment is now complete.

## Step 11: Modify Content, Prepare v2 Version

Return to the experiment directory, modify `index.html`:

    cat > index.html <<'EOF'
    <html>
      <head>
        <meta charset="utf-8">
        <title>manual release v2</title>
      </head>
      <body>
        <h1>Hello K8s CI/CD</h1>
        <p>version: v2</p>
      </body>
    </html>
    EOF

### Understanding This Step

This simulates the simplest business change.  
Subsequent actions are very similar to the first deployment, which is one of the reasons CI/CD emerged.

## Step 12: Build v2 Image and Local Verification

Execute:

    docker build -t manual-web:v2 .

Local verification:

    docker run -d --name manual-web-v2 -p 18080:80 manual-web:v2
    curl http://127.0.0.1:18080
    docker rm -f manual-web-v2

### Expected Outcomes

Output should include:

    version: v2

## Step 13: Push v2 to Harbor

Tag:

    docker tag manual-web:v2 `Yours.HarborDomain Name`/test/manual-web:v2

Push:

    docker push `Yours.HarborDomain Name`/test/manual-web:v2

Confirm `v2` exists on Harbor page.

## Step 14: Update Deployment to Use v2 Image

Execute:

    kubectl -n test set image deployment/manual-web manual-web=`Yours.HarborDomain Name`/test/manual-web:v2

### Observe Update Process Simultaneously

Recommend running three windows separately:

    kubectl -n test get deploy -w
    kubectl -n test get rs -w
    kubectl -n test get pods -w

### Key Observations in This Step

1. Deployment won't directly modify old Pods
2. A new ReplicaSet will appear
3. Old and new Pods will coexist for a period
4. Old Pods will exit gradually after new Pods are ready

This is the intuitive manifestation of rolling updates.

## Step 15: Check Rollout Status

Execute:

    kubectl -n test rollout status deployment/manual-web

### Expected Outcomes

Typically, you'll see something like:

deployment "manual-web" successfully rolled out

### Step Understanding

This indicates that from the K8s controller's perspective, the Deployment update has completed convergence.

## Step 16: Re-verify Business Content

Enter the cluster again to verify:

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

Execute:

    wget -qO- http://manual-web.test.svc.cluster.local

### Expected Phenomenon

This time you should see:

    version: v2

At this point, the second manual release is complete.

## Result Summary

After completing these steps, you have manually performed the following actions in the experiment:

1. Prepare application content
2. Write Dockerfile
3. Build local image
4. Local verify image
5. Tag image for Harbor
6. Push to Harbor
7. Write Deployment and Service
8. Deploy to K8s
9. Verify service access within the cluster
10. Rebuild after content changes
11. Re-push
12. Update Deployment image
13. Observe rolling update
14. Re-verify results

This is a minimal, real, and repeatable manual release pipeline.

## Understanding Why CI/CD is Needed Through Experiment Phenomena

Now without abstract definitions, just look at what we did.

When doing the second release, you repeated many actions:

- Change content
- Build
- Local check
- Tag
- Push
- Update image
- Rollout status
- Business verification

If your application is no longer a static page, but:

- Java service
- Frontend project
- Go API
- Multiple microservices

And release frequency increases, manually repeating these actions will increasingly lead to the following issues:

### 1) Multiple steps, prone to operational errors

For example:

- Tag written incorrectly
- Push to wrong repository
- Image address written incorrectly
- Update to wrong namespace
- Forgetting to verify results

### 2) Obvious repetitive labor

The second release is highly similar to the first, indicating many actions can be automated.

### 3) Higher version tracking requirements

You will inevitably encounter these issues later:

- Which version is currently online
- Which image tag corresponds to this release
- Which version to rollback to if problems occur
- Who performed this release

### 4) Difficult to stabilize with multi-person collaboration

When releases are no longer done by a single person but involve developers, testers, and operators, manual processes easily lose control.

Thus, the core role of CI/CD is to gradually standardize, automate, and trackable the repetitive and error-prone manual release pipeline we just went through.

## Content You Should Be Able to Explain After Completing This Experiment

After completing the experiment, it's recommended you can clearly explain the following:

Kubernetes releases not source code, but images.  
Images are first built locally, then pushed to Harbor, which serves as a unified image repository for cluster pulls.  
Deployment uses images to start Pods, Service provides a unified access entry.  
When Deployment's image changes from v1 to v2, Kubernetes triggers a rolling update, gradually creating new Pods and replacing old ones.  
A single manual release already includes build, push, deploy, and verify actions, while CI/CD automates these actions in subsequent steps.

## Common Faults and Troubleshooting

### Fault 1: Image Push Failure

Check:

    docker login your Harbor domain
    docker images | grep manual-web

Confirm:

- Harbor login successful
- Repository address correct
- Tag correct

### Fault 2: Pod Can't Pull Image

Check:

    kubectl -n test describe pod Pod name

Focus on Events.

Prioritize checking:

- Harbor address correct
- Tag exists
- containerd trusts Harbor
- Private repository authentication normal

### Fault 3: Pod Running But Content Not Updated

Check:

    kubectl -n test describe deploy manual-web
    kubectl -n test get rs
    kubectl -n test get pods
    kubectl -n test logs Pod name

Confirm:

- Deployment actually uses v2
- Harbor has real v2
- Pod actually pulled new image

### Fault 4: Rollout Stuck

Prioritize checking:

- New Pod Ready
- Image pull failure
- Application startup failure
- Pending status

Commands:

    kubectl -n test get pods
    kubectl -n test describe pod Pod name
    kubectl -n test rollout status deployment/manual-web

## Practice Requirements

Recommended to do two more practice sessions.

### Practice 1: Self-publish v3

Requirements:

- Change page to `version: v3`
- Complete the entire process without referring to the text
- Finally verify v3 in the cluster

### Practice 2: Intentionally Create a Failed Release

Requirements:

- Change Deployment image to a non-existent tag
- Observe Pod status
- Check `describe pod`
- Check `rollout status`

Through this practice, build first-level troubleshooting intuition for "release failure phenomena".

## Key Points Mastered in This Section

After completing this section, you should master:

1. How to build a minimal nginx static page image
2. How to locally verify image content
3. How to push image to Harbor
4. How to deploy image to K8s in the test namespace
5. How to access application within the cluster via Service
6. How to update Deployment image and observe rolling update
7. Why manual release pipeline evolves into CI/CD

## One-Sentence Summary

In the current experimental environment, a minimal manual release already includes image building, Harbor push, K8s deployment, rolling update, and result verification, while the value of CI/CD lies in automating this repetitive and error-prone manual release pipeline.

## Next Section

Next section enters:

02-CI-CD Overall Pipeline: Code, Build, Image, Repository, and Cluster Deployment Panorama

The next section will, based on the actions already completed in this experiment, split this pipeline into several fixed stages, clearly defining what each stage does.