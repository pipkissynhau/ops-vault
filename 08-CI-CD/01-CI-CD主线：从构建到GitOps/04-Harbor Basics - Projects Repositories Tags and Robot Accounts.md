# 04-Harbor Basics: Projects, Repositories, Tags, and Robot Accounts

## Document Notes

This article is the 4th note in the 08-CI-CD learning path.

The previous article completed two things:

1. Understanding how images are built
2. Understanding why image tags cannot long-term only use `latest`, and manually simulated a more production-like tag

This article continues the journey, entering the image repository phase, focusing on solving these actual issues:

- Why local images are insufficient
- What exactly are projects, repositories, and tags in Harbor
- How to view and confirm an image after pushing it to Harbor
- Why Robot accounts are suitable for GitLab CI/Jenkins
- How Harbor connects with K8s for private image pulling in the future

This article continues based on the current environment:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private registry
- `test` namespace as the experimental environment

## Tags

#Kubernetes #CI-CD #Harbor #MirrorRepository #MirrorTag #RobotAccountNumber #PrivateWarehouse #imagePullSecrets #I'llTakeYourNotes.

## Learning Objectives

After completing this article, you should understand:

1. Harbor's position in the entire release pipeline
2. The relationship between projects, repositories, and tags
3. Why local images must enter Harbor to be suitable for cluster-wide use
4. How to confirm image push success on the Harbor page
5. The purpose and use cases of Robot accounts
6. How Harbor connects with K8s for private image pulling in the future

## This Article's ExperimentalMain

This article continues with the image you built in the previous article, without changing scenarios.

TheMain is divided into 5 sections:

1. Confirm the difference between local and Harbor images
2. Manually push a more production-like tag to Harbor
3. Confirm the relationship between "project/repository/tag" on the Harbor page
4. Update K8s Deployment with the Harbor image
5. Understand Robot accounts and their relationship with private image pulling

## Part One: First confirm Harbor's position in the entire pipeline

First, recall the existing pipeline:

Application content  
→ Image building  
→ Image repository  
→ Cluster configuration  
→ Cluster deployment  
→ Deployment verification

Harbor's position is:

**Image repository stage**

This means:

- Docker build produces local images
- Harbor transforms these images into "cluster-unified-pullable" versions
- K8s pulls images from Harbor, not from your local machine

This understanding must be established first.

## Part Two: Confirm local and Harbor images are not the same

### Step 1: Check current local images

Execute:

    docker images | grep manual-web

If you followed the previous article's exercises, you should see tags like:

- `v3`
- `dev-a1b2c3d-101`
- `main-f6e7d8c-205`

### Step 2: Confirm image versions on the Harbor page

Log in to the Harbor page, enter the project you're currently using, e.g., `test`.

Find the repository:

- `manual-web`

Confirm which tags exist.

### Understanding to establish at this stage

Clearly distinguish:

#### Local Image

- On your current build machine
- `docker images` can see
- Only represents your local build result

#### Harbor Image

- In the Harbor registry
- Visible on the Harbor page
- Clusters ultimately pull from here

Therefore:

**Local build success ≠ Harbor having this version.**  
Only after successful push, when the tag appears on the Harbor page, K8s can pull it.

## Part Three: Review the production-style tag from the previous article and continue pushing to Harbor

This section continues using the tag strategy from the previous article, but focuses on understanding through Harbor.

### First simulate a group of image tag information

Assume this build comes from:

- Branch name: `dev`
- Commit ID: `c1d2e3f`
- Pipeline ID: `301`

### Step 1: Tag local image with production-style tag

Execute:

    docker tag manual-web:v3 manual-web:dev-c1d2e3f-301

Check local images:

    docker images | grep manual-web

### Step 2: Tag full address for Harbor

Execute:

    docker tag manual-web:v3 your-Harbor-domain/test/manual-web:dev-c1d2e3f-301

Example:

    docker tag manual-web:v3 harbor.example.com/test/manual-web:dev-c1d2e3f-301

### Step 3: Push to Harbor

Execute:

    docker push your-Harbor-domain/test/manual-web:dev-c1d2e3f-301

### Understanding to establish at this stage

This sequence actually involves two "tag" operations:

#### First tag

    docker tag manual-web:v3 manual-web:dev-c1d2e3f-301

Solves:

- Giving the local image a more production-like version identity

#### Second tag

    docker tag manual-web:v3 harbor.example.com/test/manual-web:dev-c1d2e3f-301

Solves:

- Telling Docker: where to push this image to Harbor's project, repository, and tag

### Step 4: Confirm results on the Harbor page

After logging into the Harbor page, locate:

- Project: `test`
- Repository: `manual-web`

Confirm the new tag:

- `dev-c1d2e3f-301`

## Part Four: Understand "Project/Repository/Tag" separately in Harbor

This section should not just focus on the names, but combine them with the image address you pushed earlier.

Assume you pushed:

    harbor.example.com/test/manual-web:dev-c1d2e3f-301

Break it down.

### 1) Harbor Address

    harbor.example.com

Represents the access address of the Harbor service.

### 2) Project (Project)

    test

Represents a first-level logical isolation space in Harbor.

Currently, you can understand projects as:

- A certain team
- A certain business group
- A certain environment domain
- Or a grouping of certain image categories

In your current learning environment, using `test` project is very appropriate.

### 3) Repository

    manual-web

Represents a specific image repository.

You can understand it as:

- All historical image versions of the same application are stored under this repository

In other words:

- `manual-web:v1`
- `manual-web:v2`
- `manual-web:v3`
- `manual-web:dev-c1d2e3f-301`

Essentially all belong to the same repository:

- `manual-web`

### 4) Tag

    dev-c1d2e3f-301

Represents a specific version in this repository.

### Current Understanding to Establish

In the future, when seeing an image address, you should be able to naturally break it down into:

- Harbor address
- Project
- Repository
- Tag

This is critical for troubleshooting image address errors later.

## Fifth Part: Manually Confirming "Project / Repository / Tag" Relationship on Harbor Page

This section is an important exercise, not just for viewing the page.

### Step 1: Enter Harbor Project Page

Open the Harbor page and enter:

- `test`

### Step 2: Find Repository List

Find:

- `manual-web`

### Step 3: Click into Repository Details

View all tags under this repository.

If you've done the previous sections, you might see:

- `v1`
- `v2`
- `v3`
- `dev-a1b2c3d-101`
- `main-f6e7d8c-205`
- `dev-c1d2e3f-301`

### Observation Points for This Section

Here, focus on establishing three awarenesses:

1. A single application typically corresponds to one repository
2. A repository will have multiple tags
3. Tags are identifiers for different versions of images in the repository

This step must be done by you personally clicking through the page, as the Harbor page is a direct confirmation entry for troubleshooting image versions in later work.

## Sixth Part: Actually Handing Over Harbor Images to K8s

This section reconnects Harbor with K8s.

Previously, you already knew:

- The existence of an image tag in Harbor doesn't mean K8s is using it
- K8s still needs to reference the specific image via the Deployment's image field

### Step 1: Update Deployment with a New Harbor Tag

Execute:

    kubectl -n test set image deployment/manual-web manual-web=your-Harbor-domain/test/manual-web:dev-c1d2e3f-301

### Step 2: Observe Deployment Update Process

Recommend opening three windows to observe:

    kubectl -n test get deploy -w
    kubectl -n test get rs -w
    kubectl -n test get pods -w

### Step 3: Check Rollout Status

Execute:

    kubectl -n test rollout status deployment/manual-web

### Step 4: Enter Cluster to Verify Page Content

Execute:

    kubectl -n test run curl-test --image=busybox:1.35 --rm -it -- sh

After entering, execute:

    wget -qO- http://manual-web.test.svc.cluster.local

### Current Understanding to Establish

By this point, you should reassemble the entire chain:

1. Build image locally
2. Push to Harbor
3. Form project/repository/tag in Harbor
4. Deployment references a specific Harbor tag via the image field
5. K8s nodes pull the image from Harbor
6. Deployment triggers rolling update

Thus, Harbor is not isolated; it is:

**The transit center between image building and K8s deployment.**

## Seventh Part: If K8s Fails to Pull an Image, Prioritize Harbor-Related Issues

This section is very practical for your current environment.

If your Deployment doesn't start normally after an update, common statuses include:

- `ErrImagePull`
- `ImagePullBackOff`

First execute:

    kubectl -n test get pods
    kubectl -n test describe pod Pod-name

Focus on the Events.

### Priority Troubleshooting 1: Is the Image Address Correctly Written?

Check:

- Harbor domain name
- Project name
- Repository name
- Tag

Even a single character missing will cause pull failure.

### Priority Troubleshooting 2: Does Harbor Actually Have This Tag?

Confirm on the Harbor page.

Don't rely solely on your own "I think I pushed it," manually confirm it exists on the page.

### Priority Troubleshooting 3: Does the Node Trust Harbor?

You've already handled Harbor certificate and containerd issues previously.

So if problems arise here, go back to the directions you previously troubleshooted:

- containerd's certs.d configuration
- Harbor certificate trust
- Whether the node's pull path is correct

### Priority Troubleshooting 4: Is Private Repository Authentication Available?

If the Harbor project is private, nodes may still fail to pull even if the image exists.

This is where you'll encounter `imagePullSecrets` later.

## Eighth Part: What Exactly Is a Robot Account

Now explaining Robot accounts makes sense, as you already understand how Harbor is used.

Give the most direct understanding first:

**A Robot account is a Harbor machine account for automation systems.**

### Why Need a Robot Account

Because once you enter GitLab CI/Jenkins, the pipeline will need to perform these actions automatically:

- Log in to Harbor
- Push images
- Some system synchronization of images
- Sometimes also automatic image pulling

If using a human account directly, there are several issues:

- Potential over-permissions
- Password rotation inconvenience
- Not conducive to auditing
- Not suitable for long-term automation

Thus, Harbor provides Robot accounts to:

- Be used by programs
- Have controlled permissions
- Be more suitable for pipelines and system integration

## Ninth Part: How to Understand Robot Accounts Best in This Learning Phase

In this phase, you don't need to use it in pipelines immediately, but establish the correct understanding first.

### Regular User Account

More suitable for:

- Logging into the Harbor page
- Manually viewing projects and images
- Managing repositories

### Robot Account

More suitable for:

- GitLab CI
- Jenkins
- Automated image pushing
- Automation systems accessing Harbor

### Current Understanding to Establish

In the future, when you see a pipeline needing to execute:

    docker login harbor.example.com

The account used is typically:

- A Robot account

Rather than your own administrator account or personal login account.

## Tenth Part: Simulate an "Automation Account Usage Scenario" Locally First

This section does not require you to switch accounts and perform a full experiment immediately, but you should first understand the scenario.

Assume that a pipeline in the future will perform these actions:

1. Build an image
2. Log in to Harbor
3. Push a new tag
4. Update a Deployment

In this case, Harbor must have a stable credential source for this step.

In production, a more standardized practice is usually:

- Create a Robot account for a specific project
- Assign it only the push/pull permissions needed for the current project
- Use this credential set as a secret in Jenkins / GitLab CI

Having this connection established is sufficient for now.

## Part 11: How Harbor Connects to Subsequent K8s Private Image Pulling

This point must be clearly explained upfront, as it is highly relevant to your current environment.

Harbor has two connections to K8s.

### Connection 1: Publishing Side

You or the pipeline pushes an image into Harbor.

### Connection 2: Runtime Side

K8s nodes pull an image from Harbor.

In other words, Harbor connects to:

- Image publishing
- Image runtime

So whenever you see an image in a Deployment with a Harbor address, you should naturally think of two things:

1. Whether the image has been correctly pushed to Harbor
2. Whether the nodes have the capability to pull it from Harbor

This is why Harbor is truly important in a K8s environment.

## Part 12: This Section's Practice Exercises

### Exercise 1: Push a New Production-Style Tag

Assume:

- Branch name: `test`
- Commit ID: `7c8d9e0`
- Pipeline ID: `402`

Execute:

    docker tag manual-web:v3 manual-web:test-7c8d9e0-402
    docker tag manual-web:v3 your-Harbor-domain/test/manual-web:test-7c8d9e0-402
    docker push your-Harbor-domain/test/manual-web:test-7c8d9e0-402

Then confirm on the Harbor page whether the new tag appears.

### Exercise 2: Use This New Tag to Update the Deployment

Execute:

    kubectl -n test set image deployment/manual-web manual-web=your-Harbor-domain/test/manual-web:test-7c8d9e0-402

Then observe:

    kubectl -n test rollout status deployment/manual-web

Enter the cluster to verify whether the content is normal.

### Exercise 3: Manually Answer the Following 4 Questions

1. What are projects, repositories, and tags in Harbor?
2. Why is a local image not equal to a cluster-available image?
3. Which part of Harbor does a Deployment use?
4. Why is a Robot account more suitable for pipelines than a human account?

If you can explain these 4 questions clearly, you've mastered this section.

## Content You Should Be Able to Explain After This Section

After completing this section, it's recommended that you can explain the following:

Harbor is located at the image ingestion stage, responsible for turning locally built images into a unified image version that can be pulled by the cluster.  
A complete image address can be broken down into Harbor address, project, repository, and tag.  
A single application typically corresponds to one Harbor repository, with multiple image versions under the repository with different tags.  
The image field in a Deployment references a specific tag in Harbor, and K8s nodes then pull this image from Harbor and trigger a rolling update.  
A Robot account is a machine account provided by Harbor for automation systems, suitable for GitLab CI and Jenkins to perform push/pull operations.

## Common Issues and Troubleshooting

### Issue 1: Push was successful, but not visible on Harbor page

Check first:

- Whether the project name used for pushing is correct
- Whether the project viewed on the Harbor page is correct
- Whether the repository name matches your expectation
- Whether the tag was written incorrectly

### Issue 2: Tag exists on Harbor page, but K8s cannot pull it

Check first:

- Whether the image in the Deployment is correctly written
- Whether the Harbor project is private
- Whether the nodes trust the Harbor certificate
- Whether private repository authentication is missing

### Issue 3: Why can the same image have multiple tags

Because tags are essentially multiple reference names for the same image content, and they don't necessarily represent different content.

### Issue 4: If the Robot account isn't needed now, can it be skipped

It can be skipped now, but it will be immediately needed when you reach GitLab CI / Jenkins later. Establishing this understanding is very important for this section.

## Key Points Mastered After This Section

After completing this section, you should understand:

1. Harbor's position in the release pipeline
2. The relationship between projects, repositories, and tags
3. How to confirm on the Harbor page whether an image has been truly ingested
4. The connection between Deployments and Harbor images
5. The role and use cases of Robot accounts
6. How Harbor connects to K8s private image pulling

## One-Sentence Summary

Harbor's core role is to transform locally built images into a unified, manageable, and trackable image version that can be pulled by K8s nodes, with projects, repositories, tags, and Robot accounts collectively forming the foundation of this repository management system.

## Next Section

Next section will enter:

05-GitLab CI Basics: .gitlab-ci.yml, Stage, Job, and Variables

The next section will begin mapping the manual actions performed in previous sections to GitLab CI, showing you:

- Which actions will be automated by the pipeline
- What `.gitlab-ci.yml` actually controls
- Which parts of Stage, Job, and variables correspond to in the current learning path