# 01-Why K8s Needs CI-CD: From Manual Deployment to Cloud-Native Application Delivery

## Document Description

This article is the first note in the 08-CI-CD learning pathway.

Based on the existing experimental environment, this article aims to provide an intuitive understanding of the Kubernetes application delivery process through a minimal viable deployment experiment. The focus is not on tool definitions but on demonstrating the entire manual deployment process and explaining why CI/CD systems are later introduced.

The default environment for this article includes:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private repository
- The `test` namespace as the experimental environment

## Tags

#Kubernetes #CI-CD #Harbor #Deployment #Service #Rollout #Image Deployment #Practical Notes

## Learning Objectives

Complete a minimal manual deployment and then perform a version update.

Upon completion, you should understand the following:

1. That Kubernetes ultimately runs on images, not local source code files.
2. The role of Harbor in the deployment process.
3. The steps involved in a typical manual deployment.
4. How a Deployment updates an image and triggers a rollout.
5. Why these repetitive tasks are later incorporated into CI/CD workflows.

## Steps of This Experiment

Page file → Build image → Push to Harbor → Deploy to K8s → Verify access within the cluster → Update content and re-deploy → Monitor the rollout process

## Experimental Preparation

### 1) Check Cluster Status

Run:

    kubectl get nodes
    kubectl get ns

Ensure the `test` namespace exists. If not, create it:

    kubectl create namespace test

### 2) Verify Access to Harbor and Login

Run:

    docker login your-Harbor-domain

For example:

    docker login harbor.example.com

### 3) Create an Experimental Directory

Run:

    mkdir -p ~/08-ci-cd/01-manual-release
    cd ~/08-ci-cd/01-manual-release

All subsequent files will be stored here.

## Step 1: Prepare the v1 Page File

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

### Understanding This Step

The `index.html` file represents the “application content.”  
Any version updates will essentially involve changing the content of this file, which is then rebuilt into an image and re-deployed.

## Step 2: Write a Dockerfile

Create `Dockerfile`:

    cat > Dockerfile <<'EOF'
    FROM nginx:1.27
    COPY index.html /usr/share/nginx/html/index.html
    EXPOSE 80
    EOF

### Understanding This Step

This step involves “packing the application into an image.”

The details are as follows:

- The base image is nginx.
- The local page file is copied into the image.
- When the container starts, nginx serves the static pages.

For now, we won’t include Java, Maven, Jenkins, or other components; instead, we will focus on establishing the minimum deployment process.

## Step 3: Build the Local Image

Run:

    docker build -t manual-web:v1 .

View the image:

    docker images | grep manual-web

### Expected Result

You should see something like:

    manual-web    v1

### Understanding This Step

The resulting image is **local** and exists only on the machine where it was built. It is not yet a shared resource for the entire cluster.

## Step 4: Verify the Image Content Locally

Run:

    docker run -d --name manual-web-v1 -p 18080:80 manual-web:v1

View the page:

    curl http://127.0.0.1:18080

The expected output should be:

    Hello K8s CI/CD
    version: v1

After verification, delete the container:

    docker rm -f manual-web-v1

### Understanding This Step

This step serves as a self-check for the image. It ensures that there are no issues with the image itself before proceeding to the next steps where it will be stored in Harbor and deployed to K8s.

## Step 5: Add Harbor Tags to the Image

Modify the image name to include the full Harbor address format:

    docker tag manual-web:v1 your-Harbor-domain/test/manual-web:v1

For example:

    docker tag manual-web:v1 harbor.example.com/test/manual-web:v1

### Understanding This Step

This step adds```markdown
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

This step simulates the simplest business change.  
The subsequent actions are very similar to those in the first release, which is one of the practical reasons for the emergence of CI/CD.

## Step 12: Build the v2 Image and Verify It Locally

Execute:

    docker build -t manual-web:v2 .

Local verification:

    docker run -d --name manual-web-v2 -p 18080:80 manual-web:v2
    curl http://127.0.0.1:18080
    docker rm -f manual-web-v2

### Expected Outcome

The output should include:

    version: v2

## Step 13: Push the v2 Image to Harbor

Tag the image:

    docker tag manual-web:v2 your-Harbor-domain/test/manual-web:v2

Push it:

    docker push your-Harbor-domain/test/manual-web:v2

Confirm on the Harbor page that `v2` exists.

## Step 14: Update the Deployment to Use the v2 Image

Execute:

    kubectl -n test set image deployment/manual-web manual-web=your-Harbor-domain/testmanual-web:v2

### Observing the Update Process Simultaneously

It is recommended to open three separate windows to execute the following commands:

    kubectl -n test get deploy -w
    kubectl -n test get rs -w
    kubectl -n test get pods -w

### Key Points to Observe in This Step

1. The Deployment will not directly “replace the old Pod”.
2. A new ReplicaSet will be created.
3. Both the old and new Pods will exist for a period of time.
4. Only after the new Pod is Ready will the old Pod gradually exit.

This is the direct manifestation of rolling updates.

## Step 15: Check the Rollout Status

Execute:

    kubectl -n test rollout status deployment/manual-web

### Expected Outcome

You should see something similar to:

    deployment "manual-web" successfully rolled out

### Understanding This Step

This indicates that, from the perspective of the K8s controller, this Deployment update has completed convergence.

## Step 16: Verify the Business Content Again

Verify inside the cluster again:

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

Execute:

    wget -qO- http://manual-web.test.svc.cluster.local

### Expected Outcome

This time, you should see:

    version: v2

With this, the second manual release is completed.

## Summary of Results

After completing the above steps, you have manually performed the following actions in this entire experiment:

1. Prepared the application content.
2. Written a Dockerfile.
3. Built a local image.
4. Verified the image locally.
5. Tagged the image for Harbor.
6. Pushed it to Harbor.
7. Written Deployment and Service definitions.
8. Deployed it to K8s.
9. Verified service access inside the cluster.
10. Rebuilt after making changes.
11. Pushed again.
12. Updated the Deployment image.
13. Observed the rolling update process.
14. Verified the results again.

This represents a minimal, realistic, and repeatable manual release process.

## Understanding Why CI/CD is Needed Based on Experimental Observations

Without delving into abstract definitions for now, just look at what we have done recently.

During the second release, you repeated many steps:

- Making changes.
- Building the image.
- Locally checking it.
- Tagging it.
- Pushing it.
- Updating the image.
- Checking the rollout status.
- Verifying the business results.

If your application in the future is not just a static web page but something like:

- A Java service.
- A front-end project.
- A Go API.
- Multiple microservices,

and if the release frequency increases, manually repeating these steps will likely lead to the following problems:

### 1) Many Steps and High Risk of Operational Errors

For example:

- Making a mistake in tagging the image.
- Pushing it to the wrong repository.
- Entering the incorrect image address.
- Updating it to the wrong namespace.
- Forgetting to verify the results.

### 2) Obvious Repetitive Work

Since the second release was very similar to the first, it means that many of these steps could actually be automated.

### 3) Increasing Requirements for Version Tracking

You will inevitably face these issues in the future:

- Which version is currently running online?
- Which image tag corresponds to this release