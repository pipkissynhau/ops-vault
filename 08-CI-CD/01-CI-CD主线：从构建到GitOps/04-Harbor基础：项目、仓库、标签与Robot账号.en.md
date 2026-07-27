# 04-Harbor Basics: Projects, Repositories, Tags, and Robot Accounts

## Documentation Overview

This document is the fourth note in the 08-CI-CD learning pathway.

In the previous article, two key tasks were completed:

1. Understanding how images are constructed.
2. Realizing why using `latest` as a tag for images should not be done indefinitely, and manually simulating production-style tags.

This article builds on these concepts by delving into the realm of image repositories, focusing on solving the following practical issues:

- Why local images alone are insufficient.
- What exactly are projects, repositories, and tags in Harbor.
- How to check and confirm that an image has been successfully pushed to Harbor.
- Why Robot accounts are suitable for use with GitLab CI/Jenkins.
- How to integrate Harbor with K8s for private image retrieval.

This article continues to use the existing environment:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private repository
- The `test` namespace as the experimental environment

## Tags

#Kubernetes #CI-CD #Harbor #Image Repository #Image Tag #Robot Account #Private Repository #imagePullSecrets #Practical Notes

## Learning Objectives

After completing this article, you should be able to:

1. Understand the role of Harbor in the entire deployment pipeline.
2. Grasp the relationship between projects, repositories, and tags.
3. Recognize why local images must be stored in Harbor for unified cluster use.
4. Know how to confirm successful image pushes on the Harbor interface.
5. Comprehend the purpose and applicable scenarios of Robot accounts.
6. Understand how to integrate Harbor with K8s for private image retrieval.

## Main Experimental Approach

This article builds upon the images you created in the previous article, without changing the experimental setup.

The main process consists of five steps:

1. Differentiating between local images and Harbor images.
2. Manually pushing a production-style tag to Harbor.
3. Confirming the relationship between "project/repository/tag" on the Harbor interface.
4. Using Harbor images to update a K8s Deployment.
5. Understanding the role of Robot accounts in subsequent private image retrieval.

## Part 1: Understand Harbor's Position in the Entire Pipeline

Let's revisit the previously learned pipeline:

Application Content  
→ Image Construction  
→ Image Storage in Harbor  
→ Cluster Configuration  
→ Cluster Deployment  
→ Release Verification

Harbor plays a crucial role during the **image storage phase**. In other words:

- The `docker build` command produces local images.
- Harbor is responsible converting these local images into versions that can be uniformly pulled by the cluster.
- K8s subsequently retrieves images from Harbor, not from your local machine.

This understanding is essential at this stage.

## Part 2: Recognize That Local Images and Harbor Images Are Not the Same

### Step 1: View Current Local Images

Run:

    docker images | grep manual-web

If you followed the exercises in the previous article, you should see tags like these:

- `v3`
- `dev-a1b2c3d-101`
- `main-f6e7d8c-205`

### Step 2: Check Image Versions on the Harbor Interface

Log in to the Harbor interface and navigate to your current project, for example, `test`.

Find the repository:

- `manual-web`

Check which tags are available.

### Key Points to Understand Here

It is crucial to distinguish between:

#### Local Images

- Located on your local build machine.
- Visible through `docker images`.
- Represent only your local build results.

#### Harbor Images

- Stored in the Harbor repository.
- Visible on the Harbor interface.
- These are the images that will actually be pulled by the cluster.

Therefore:

**A successful local build does not mean that the corresponding version exists in Harbor.** Only after a successful push will the tag appear on the Harbor interface, allowing K8s to retrieve it later.

## Part 3: Review the Production-style Tags Used in the Previous Article and Push Them to Harbor Again

This part continues using the same tagging approach discussed in the previous article, but focuses on understanding how it works within Harbor.

### Step 1: Simulate a Set of Image Tag Information

Assume this build is from:

- Branch name: `dev`
- Commit ID: `c1d2e3f`
- Pipeline ID: `301`

### Step 1: Apply Production-style Tags to Local Images

Run:

    docker tag manual-web:v3 manual-web:dev-c1d2e3f-301

Check local images:

    docker images | grep manual-web

### Step 2: Apply the Complete Tag to Harbor

Run:

    docker tag manual-web:v3 harbor.example.com/test/manual-web:dev-c1d2e3f-303. Projects, repositories, and tags are created within Harbor.
4. During deployment, the `image` field references a specific tag in Harbor.
5. K8s nodes pull this image from Harbor.
6. The Deployment triggers rolling updates.

Therefore, Harbor is not an isolated system; it serves as:

**The intermediary between image building and K8s deployment.**

## Section 7: If K8s fails to pull an image, start by checking Harbor-related issues

This section is very practical for your current environment.

If your Pod does not start up properly after updating a Deployment, common errors include:

- `ErrImagePull`
- `ImagePullBackOff`

First, execute the following commands:

    kubectl -n test get pods
    kubectl -n test describe pod PodName

Pay special attention to the Events section.

### Priority Check 1: Is the image address correct?

Verify the following:

- Harbor domain name
- Project name
- Repository name
- Tag

Even a single missing character can cause the pull to fail.

### Priority Check 2: Does the tag actually exist in Harbor?

Confirm this on the Harbor page. Don’t just rely on the assumption that you pushed it; manually verify its presence.

### Priority Check 3: Does the node trust Harbor?

You have already dealt with the issue of Harbor certificates and containerd. If there is a problem here, return to the steps you previously checked:

- `containerd`’s certs.d configuration
- Harbor certificate authentication
- Whether the node’s pull path is correct

### Priority Check 4: Is private repository authentication in place?

If the Harbor project is private, the node may still fail to pull the image due to lack of permissions. This is where the `imagePullSecrets` come into play.

## Section 8: What exactly is a Robot account?

Now it’s appropriate to discuss Robot accounts since you already understand how Harbor is used.

Here’s the most straightforward explanation:

**A Robot account is a dedicated account in Harbor for use by automated systems.**

### Why are Robot accounts needed?

Once you move on to GitLab CI/Jenkins, pipelines will need to perform these actions automatically:

- Log in to Harbor
- Push images
- Synchronize images with certain systems
- Occasionally pull images automatically

Using a human account at this point would present several issues:

- Excessive permissions
- Inconvenience with password rotation
- Difficulty in auditing
- Unsuitability for long-term automation

Therefore, Harbor provides Robot accounts to:

- Enable use by programs
- Maintain controllable permissions
- Better suit pipelines and system integration

## Section 9: How to understand Robot accounts at this stage

You don’t need to use them in your pipelines immediately, but it’s important to establish a correct understanding now.

### Regular user accounts

Are more suitable for:

- Logging into the Harbor page
- Manually viewing projects and images
- Managing repositories

### Robot accounts

Are more suitable for:

- GitLab CI
- Jenkins
- Automatically pushing images
- Automated system access to Harbor

### Understanding required at this stage

In the future, when you see a pipeline command like:

    docker login harbor.example.com

the account used should typically be a Robot account, rather than your own administrator or personal login account.

## Section 10: Simulate an “automated account usage scenario” locally first

You don’t need to switch accounts and perform the entire experiment now, but you should understand the scenario.

Imagine that a future pipeline needs to perform these actions:

1. Build an image
2. Log in to Harbor
3. Push a new tag
4. Update a Deployment

In this case, Harbor must have a stable source of credentials for this step.

In production, a more standardized approach would be:

- Create a Robot account for the project
- Grant it only the necessary push/pull permissions for that project
- Use these credentials as secrets in Jenkins/GitLab CI

For now, just establishing this connection is sufficient.

## Section 11: How Harbor integrates with subsequent K8s private image pulls

This point must be clarified in advance because it is closely related to your current environment.

Harbor has two connections to K8s:

### Connection 1: Release side

You or the pipeline pushes the image into Harbor.

### Connection 2: Runtime side

K8s nodes pull the image from Harbor.

In other words, Harbor connects both:

- Image release
- Image runtime

Therefore, whenever you see an `image` field in a Deployment that points to a Harbor address, you should consider two things:

1. Has the image been correctly pushed to Harbor?
2. Does the node have the capability to pull it down from Harbor?

This is the true significance of Harbor in a K8s environment.

## Section 12: This chapter’s exercises

### Exercise 1: Push a new production-style tag again

Assume:

- Branch