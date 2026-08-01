# 02-CI-CD Overall Pipeline: Breaking a Manual Deployment into Code, Build, Image, Repository, Configuration, and Cluster Deployment Stages

## Document Notes

This is the second note in the 08-CI-CD learning path.

The previous article completed a minimal manual deployment experiment:

Page file  
→ Build image  
→ Push to Harbor  
→ Deploy to K8s  
→ Modify content and redeploy  
→ Observe rolling update

This article doesn't introduce new tools, but focuses on breaking down the actions from the previous experiment into a clear delivery pipeline, and aligning each segment through small experiments.

The goal isn't to memorize concepts, but to achieve these two things:

1. Know the fixed 6 stages a deployment goes through
2. Be able to accurately map your own experimental actions to these stages

When learning GitLab CI, Jenkins, Helm, and Argo CD later, this pipeline will be repeatedly used.

The environment assumed in this article is:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private registry
- `test` namespace as the experimental environment

## Tags

#Kubernetes #CI-CD #Harbor #Deployment #MirrorBuild #ApplyDelivery #I'llTakeYourNotes. #ReleaseLinks

## Learning Objectives for This Article

After completing this article, you should master:

1. Ability to break a deployment into fixed 6 segments
2. Ability to clearly explain the input and output of each segment
3. Ability to verify each segment using real files and commands from your environment
4. Understand which segments GitLab CI/Jenkins will automate later
5. Establish the awareness of "checking which pipeline segment failed first" for troubleshooting

## First, Establish the Fixed Pipeline

At this learning stage, fix the delivery pipeline as the following 6 segments:

1. Application Content Stage
2. Image Build Stage
3. Image Repository Stage
4. Cluster Configuration Stage
5. Cluster Deployment Stage
6. Deployment Verification Stage

You can first map the actions from the previous article to these 6 segments:

- `index.html` belongs to Application Content Stage
- `docker build` belongs to Image Build Stage
- `docker push` belongs to Image Repository Stage
- `manual-web.yaml` belongs to Cluster Configuration Stage
- `kubectl apply` / `kubectl set image` belong to Cluster Deployment Stage
- `wget` / `rollout status` belong to Deployment Verification Stage

This article will verify each of these 6 segments one by one.

## Experiment Preparation

Continue using the directory from the previous experiment:

    cd ~/08-ci-cd/01-manual-release

If the directory doesn't exist, it means the previous experiment wasn't completed. It's recommended to complete the previous article first before proceeding.

## Part 1: Application Content Stage

The core question for this segment is:

**What exactly are you deploying.**

In the previous experiment, the minimal application content deployed was:

    index.html

Check the file:

    cat index.html

If you've already deployed to v2, you should see:

    version: v2

### Input and Output for This Segment

#### Input

- Business content
- Page file
- Source code
- Configuration content

In the current experiment, it's `index.html`

#### Output

- A clear "content to be deployed"

In the current experiment, it's the version difference in page content, such as:

- `version: v1`
- `version: v2`

### Understanding to be Established in This Section

This segment isn't Kubernetes or Harbor yet.  
It's simply answering:

> What exactly is the content I'm deploying this time.

If it's later changed to a Java project, this corresponds to source code and code changes.  
But in this stage, use static pages to fulfill this role first.

## Small Exercise for This Segment

Modify the page content to a new version you can clearly identify, such as:

    cat > index.html <<'EOF'
    <html>
      <head>
        <meta charset="utf-8">
        <title>manual release v2.1</title>
      </head>
      <body>
        <h1>Hello K8s CI/CD</h1>
        <p>version: v2.1</p>
      </body>
    </html>
    EOF

After execution, check:

    cat index.html

### Observation Points for This Segment

You'll notice:

- Only the local file was changed
- The image in Harbor hasn't changed
- The Pod in K8s hasn't changed
- The cluster access result hasn't changed

This indicates:

**Changing files alone doesn't equal completing a deployment.**

## Part 2: Image Build Stage

The core question for this segment is:

**How to turn application content into a Kubernetes runnable image.**

In the current experiment, this corresponds to:

    docker build -t manual-web:v2.1 .

Execute the build:

    docker build -t manual-web:v2.1 .

Check local images:

    docker images | grep manual-web

### Input and Output for This Segment

#### Input

- Application content files
- Dockerfile

In the current experiment, it's:

- `index.html`
- `Dockerfile`

#### Output

- Local image

In the current experiment, it's:

- `manual-web:v2.1`

### Understanding to be Established in This Section

After this segment, you get:

**A locally runnable image**

But it's still just a result on the current machine and can't be uniformly used by the cluster yet.

## Small Exercise for This Segment

Verify the image content by running it locally:

    docker run -d --name manual-web-v21 -p 18080:80 manual-web:v2.1
    curl http://127.0.0.1:18080
    docker rm -f manual-web-v21

### Observation Points for This Segment

You should see:

    version: v2.1

This indicates:

- Application content has entered the image
- The image itself is working
- But it's still unrelated to Harbor and K8s

In other words:

**The end mark of the image build stage is that the local image can run normally.**

## Part 3: Image Repository Stage

The core question for this segment is:

**How to make this image become a deliverable that the cluster can uniformly pull.**

First tag the Harbor repository:

    docker tag manual-web:v2.1 Yours.HarborDomain Name/test/manual-web:v2.1

Then push:

    docker push Yours.HarborDomain Name/test/manual-web:v2.1

Example:

    docker tag manual-web:v2.1 harbor.example.com/test/manual-web:v2.1
    docker push harbor.example.com/test/manual-web:v2.1

### Input and Output for This Segment

#### Input /think
</think>

#### Input

- Local image
- Harbor repository address
- Image tag

In the current experiment, it's:

- `test/manual-web:v2.1`
- `manual-web`
- `v2.1`

#### Output

- Image in Harbor repository

In the current experiment, it's:

- `ImagePullBackOff`

### Understanding to be Established in This Section

After this segment, you get:

**An image that can be uniformly pulled by the cluster**

It's no longer just a local result but becomes a shared deliverable.

## Small Exercise for This Segment

Push the image to Harbor and verify:

    docker tag manual-web:v2.1 http://manual-web.test.svc.cluster.local/test/manual-web:v2.1
    docker push http://manual-web.test.svc.cluster.local/test/manual-web:v2.1

### Observation Points for This Segment

You should see:

- The image has been successfully pushed to Harbor
- The image in Harbor matches the local one

This indicates:

**The end mark of the image repository stage is that the image is successfully stored in the repository.**

## Part 4: Cluster Configuration Stage

The core question for this segment is:

**How to make the cluster know where to find the image.**

In the current experiment, this corresponds to:

    kubectl apply -f deployment.yaml

Check the deployment:

    kubectl get deployments

### Input and Output for This Segment

#### Input

- Image repository address
- Image tag
- Cluster configuration file

In the current experiment, it's:

- `manual-web.yaml`
- `v1`
- `v2`

#### Output

- Cluster configuration

In the current experiment, it's:

- `v2.1`

### Understanding to be Established in This Section

After this segment, you get:

**Cluster configuration that knows where to find the image**

It's no longer just a local image but becomes a cluster-aware configuration.

## Small Exercise for This Segment

Verify the cluster configuration:

    kubectl get deployments

### Observation Points for This Segment

You should see:

- The deployment has been created
- The image address in the deployment matches the one in Harbor

This indicates:

**The end mark of the cluster configuration stage is that the cluster knows where to find the image.**

## Part 5: Cluster Deployment Stage

The core question for this segment is:

**How to deploy the image to the cluster.**

In the current experiment, this corresponds to:

    kubectl apply -f deployment.yaml

Check the pod status:

    kubectl get pods

### Input and Output for This Segment

#### Input

- Cluster configuration
- Image repository address
- Image tag

In the current experiment, it's:

- `v2`
- `v2.1`
- `set image`

#### Output

- Running pods in the cluster

In the current experiment, it's:

- `kubectl apply`

### Understanding to be Established in This Section

After this segment, you get:

**Running pods in the cluster**

It's no longer just a local configuration but becomes a cluster-deployed service.

## Small Exercise for This Segment

Verify the deployment:

    kubectl get pods

### Observation Points for This Segment

You should see:

- The pod has been created
- The pod is running
- The pod is using the image from Harbor

This indicates:

**The end mark of the cluster deployment stage is that the image is successfully running in the cluster.**

## Part 6: Deployment Verification Stage

The core question for this segment is:

**How to confirm the deployment is successful.**

In the current experiment, this corresponds to:

    curl http://127.0.0.1:18080

Check the result:

    curl http://127.0.0.1:18080

### Input and Output for This Segment

#### Input

- Cluster deployment
- Image repository address
- Image tag

In the current experiment, it's:

- `rollout status`
- `version: v2.1`
- `kubectl -n test rollout status deployment/manual-web`

#### Output

- Deployment verification result

In the current experiment, it's:

- `wget -qO- http://manual-web.test.svc.cluster.local`

### Understanding to be Established in This Section

After this segment, you get:

**Confirmation that the deployment is successful**

It's no longer just a cluster deployment but becomes a verified operational result.

## Small Exercise for This Segment

Verify the deployment:

    curl http://127.0.0.1:18080

### Observation Points for This Segment

You should see:

- The content matches the latest version
- The deployment is successful

This indicates:

**The end mark of the deployment verification stage is that the deployment is confirmed to be successful.**

## Summary

This article has verified the 6 fixed stages of a deployment:

1. Application Content Stage
2. Image Build Stage
3. Image Repository Stage
4. Cluster Configuration Stage
5. Cluster Deployment Stage
6. Deployment Verification Stage

By understanding these stages, you can:

- Identify which stage a deployment failure occurs in
- Map your own experimental actions to these stages
- Build a foundation for understanding automated CI/CD tools like GitLab CI and Jenkins

- Local Image  
- Harbor Address  
- Project Name  
- Repository Name  
- Tag Name  

#### Output  

- New version of image existing in Harbor  

In the current experiment:  

- `test/manual-web:v2.1`  

### Understanding to Establish in This Section  

After this section, the image truly transitions from:  

- Local result  

to:  

- Unified image version that the cluster can pull  

Therefore, Harbor's role in the chain is:  

**Image central storage point**  

## Small Exercise in This Section  

Manually confirm on the Harbor page:  

- Whether `manual-web` repository appears  
- Whether `v2.1` tag exists  
- Whether it matches the push address  

### Observation Points in This Section  

Here, we need to deliberately establish an awareness:  

- Having an image locally does not mean the cluster can use it  
- Only when Harbor actually contains this tag can the Deployment possibly successfully pull it  

This is crucial for troubleshooting `ImagePullBackOff` later.  

## Part Four: Cluster Configuration Phase  

The core issue in this section is:  

**Which image, and in what way, should run in K8s.**  

Check the YAML from the previous article:  

    cat manual-web.yaml  

### Inputs and Outputs in This Section  

#### Inputs  

- Image address  
- Replica count  
- Container port  
- Resource definition  

#### Outputs  

- A configuration file that can be submitted to K8s  

In the current experiment, this configuration is:  

- `manual-web.yaml`  

### Understanding to Establish in This Section  

Here, pay special attention:  

**YAML is not the application content itself, nor is it the image itself.**  

It describes:  

- Which image to use  
- How many replicas to start  
- Which tag to use  
- How the Service selects the Pod  

Therefore, the configuration phase and image phase are two separate lines:  

- The image phase solves "what to run"  
- The configuration phase solves "how to run"  

## Small Exercise in This Section  

Check the `image` field in the current YAML:  

    grep image manual-web.yaml  

If it's still the old version, such as `v1` or `v2`, manually edit it to `v2.1`.  

Example:  

    sed -i 's#manual-web:v2#manual-web:v2.1#g' manual-web.yaml  

If your current file isn't `v2`, manually open and edit it; there's no need to strictly follow the command.  

After modification, confirm again:  

    grep image manual-web.yaml  

### Observation Points in This Section  

You will notice:  

- Even if Harbor already has `v2.1`  
- As long as the YAML still references the old version  
- The cluster will not automatically use the new image  

This indicates:  

**Image being stored in the repository does not mean the cluster has already been deployed.**  

## Part Five: Cluster Deployment Phase  

The core issue in this section is:  

**How to truly deliver this configuration to Kubernetes.**  

In the current experiment, there are two methods.  

### Method One: Directly apply YAML  

Execute:  

    kubectl apply -f manual-web.yaml  

### Method Two: Directly update Deployment image  

Execute:  

    kubectl -n test set image deployment/manual-web manual-web=your Harbor domain/test/manual-web:v2.1  

Both methods belong to the "cluster deployment phase."  

### Inputs and Outputs in This Section  

#### Inputs  

- YAML configuration  
- Or an image update command  

#### Outputs  

- Deployment begins updating  
- New Pod starts creating  
- Old Pod is gradually replaced  

### Understanding to Establish in This Section  

This section is not about building or pushing, but about:  

**Informing Kubernetes of the target version**  

Next, Kubernetes will handle:  

- Creating new ReplicaSet  
- Pulling new image  
- Creating new Pod  
- Rolling out replacement of old Pod  

## Small Exercise in This Section  

Recommended to use `set image` to update once, as the command is most intuitive:  

    kubectl -n test set image deployment/manual-web manual-web=your Harbor domain/test/manual-web:v2.1  

Then open three windows to observe:  

    kubectl -n test get deploy -w  
    kubectl -n test get rs -w  
    kubectl -n test get pods -w  

### Observation Points in This Section  

You should focus on seeing:  

1. Deployment does not directly modify old Pod  
2. A new ReplicaSet will appear  
3. New and old Pods will coexist for a period of time  
4. Old Pod exits only after new Pod is Ready  

This indicates:  

**The essence of Kubernetes deployment phase is driving rolling updates.**  

## Part Six: Deployment Verification Phase  

The core issue in this section is:  

**How to confirm that this deployment was truly successful.**  

Do not only check `kubectl apply` success, nor only check if Pod is Running.  

In the current experiment, at least two verification methods should be performed.  

### Verification 1: Check rollout status  

Execute:  

    kubectl -n test rollout status deployment/manual-web  

### Verification 2: Check actual business content  

Execute:  

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh  

After entering, execute:  

    wget -qO- http://manual-web.test.svc.cluster.local  

### Inputs and Outputs in This Section  

#### Inputs  

- Current Deployment status  
- Current Pod status  
- Service access result  

#### Outputs  

- Judgment on whether the deployment was truly completed  

### Understanding to Establish in This Section  

Here, we need to distinguish two levels:  

#### First Level: Kubernetes Controller Perspective  

`rollout status` success indicates:  

- The update process has converged  
- New Pod has started as expected  

#### Second Level: Business Perspective  

Seeing `version: v2.1` on the actual page indicates:  

- The new version content has truly taken effect externally  

Therefore:  

**The deployment verification phase is not over just by seeing a command succeed, but by checking cluster status and business results.**  

## Small Exercise in This Section  

After completing, record the following two outputs:  

1. Output of `kubectl -n test rollout status deployment/manual-web`  
2. Output of `wget -qO- http://manual-web.test.svc.cluster.local`  

### Observation Points in This Section  

Only when you simultaneously see:  

- Rollout success  
- Page content is indeed v2.1  

Can you consider this deployment truly completed.  

## Reconnecting the Entire Chain  

Now, reconnect your previous actions using the fixed chain:  

### 1) Application Content Phase  

You modified:  

- `index.html`  

### 2) Image Build Phase  

You executed:  

- `docker build -t manual-web:v2.1 .`  

### 3) Image Storage Phase  

You executed:  

- `docker tag ...`  
- `docker push ...` /think

### 4) Cluster Configuration Phase

You checked or modified:

- `manual-web.yaml`
- The `image` field within it

### 5) Cluster Deployment Phase

You executed:

- `kubectl apply -f ...`
- Or `kubectl set image ...`

### 6) Release Verification Phase

You executed:

- `kubectl rollout status ...`
- `wget -qO- http://manual-web.test.svc.cluster.local`

Now you've broken down the entire release pipeline into clear steps.

## Now Revisiting: What GitLab CI / Jenkins Will Automate in the Future

At this stage, don't write GitLab CI / Jenkins configurations yet. First, map them to the actions you've already performed manually.

The core of future pipeline automation will be these fixed steps:

### What GitLab CI / Jenkins Will Automate

1. Pull code
2. Build image
3. Tag image
4. Push to Harbor
5. Update configuration or execute deployment command
6. Perform basic verification

You don't need to rush to learn their syntax yet.  
Master this manual workflow thoroughly first, then look at pipelines - it will no longer feel abstract.

## Content You Should Be Able to Explain After This Section

After completing this section, you should be able to explain the following:

A single release can be divided into 6 fixed phases: application content, image building, image repository, cluster configuration, cluster deployment, and release verification.  
After application content changes, first build a local image via Dockerfile, then push it to Harbor.  
After a new version image exists in Harbor, you need to inform Kubernetes of which version to use via YAML or set image.  
When Deployment receives a new image, it will trigger rolling update, and finally confirm successful deployment through rollout status and actual access results.  
What GitLab CI and Jenkins will do next is essentially automate these fixed phases.

## Common Failures and Link Troubleshooting Methods

Starting from this section, it's recommended to locate issues by "which phase failed".

### Failure 1: Local build fails

Indicates the issue is in:

- Image building phase

Prioritize checking:

- Dockerfile
- index.html
- Build context

### Failure 2: Push fails

Indicates the issue is in:

- Image repository phase

Prioritize checking:

- Harbor login
- Repository address
- Tag
- Permissions

### Failure 3: Pod cannot pull image

Indicates the issue is in:

- Image repository phase and cluster deployment phase

Prioritize checking:

- Whether the image actually exists in Harbor
- Whether the image address is correct
- Whether containerd trusts Harbor
- Whether private repository authentication is normal

### Failure 4: Rollout gets stuck

Usually indicates the issue is in:

- Cluster deployment phase

Prioritize checking:

- Whether new Pod is Ready
- Whether image pull failed
- Whether container startup failed
- Whether it's Pending

### Failure 5: Rollout succeeds but page still shows old version

Usually indicates the issue is in:

- Release verification phase or configuration phase

Prioritize checking:

- The image actually referenced by Deployment
- Whether Service accesses the correct Pod
- Whether you actually pushed the new version image

## Practice Requirements

It's recommended to independently complete the following two exercises.

### Exercise 1: Re-release v2.1 as v2.2

Requirements:

- Modify `index.html`
- Build new image
- Push to Harbor
- Update Deployment
- Verify page results

Goal:

- Be able to fully walk through the 6-phase pipeline without referring to the text

### Exercise 2: Intentionally create an error in one phase and determine which phase it belongs to

You can choose any error, such as:

- Push to a non-existent Harbor address
- Deployment references a non-existent tag
- Keep old image in YAML without modification

Then answer:

1. Which phase is the error in?
2. What is the phenomenon?
3. What command should be checked first?

This exercise is crucial for future troubleshooting.

## Key Takeaways from This Section

After completing this section, you should master:

1. How to break down a single release into fixed 6 phases
2. How to distinguish between image building, image repository, cluster configuration, cluster deployment, and release verification
3. How to classify your actual executed commands into these 6 phases
4. Which phases will be automated by GitLab CI / Jenkins in the future
5. When encountering failures, first locate the issue by phase

## One-Sentence Summary

A Kubernetes release is not a single command, but a fixed pipeline composed of application content, image building, image repository, cluster configuration, cluster deployment, and release verification. The core of future CI/CD will be automating this pipeline.

## Next Section

Next section will enter:

03-Container Image Building Basics: From Java Program to Docker Image

The next section will continue with hands-on guidance, focusing on separating the "image building phase" to let you see the relationship between Dockerfile, build, tag, and push through a minimal Java/static application example.