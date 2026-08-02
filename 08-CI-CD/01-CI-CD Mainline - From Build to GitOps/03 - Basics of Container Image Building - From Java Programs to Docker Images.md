# 03 - Basics of Container Image Building: From Java Programs to Docker Images

## Document Description

This article is the third note in the 08-CI-CD learning pathway.

Following the approach used in the previous two articles, this text will use the existing experimental environment to demonstrate what actually happens during the "image building phase" through hands-on operations. It will also introduce methods for designing image tags that are more suitable for production scenarios at appropriate points.

The default environment for this article is as follows:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private repository
- The `test` namespace is used as the experimental environment

This article does not directly introduce Jenkins, GitLab CI, or Helm; instead, it focuses on clarifying the following main process:

Application content  
→ Dockerfile  
→ Local image  
→ Image tag  
→ Tracable image version in Harbor

## Tags

#Kubernetes #CI-CD #Docker #Harbor #Image Building #Dockerfile #ImageTag #commitid #pipelineid #Practical Notes

## Learning Objectives

After completing this article, you should be able to:

1. Understand what the inputs and outputs of the image building phase are.
2. Comprehend the role of Dockerfiles in image building.
3. Realize why local file changes require a rebuild to create a new image.
4. Recognize why using `latest` as an image tag for a long time is not recommended.
5. Understand what each of branch names, commit IDs, and pipeline IDs addresses.
6. Manually simulate a production-style image tagging method.
7. Explain how a specific image version was created.

## Main Experimental Process of This Article

This article is divided into two parts:

### Part 1: How Images Are Built

Continue the experiment using a minimal static web page image, focusing on observing:

- Content modification
- Image building
- Local verification

### Part 2: Designing Tags After Image Building Is Complete

Use methods closer to production practices to assign different types of tags to the same image and understand what each tag is suitable for.

## Experimental Preparation

Continue using the experimental directory from the previous two articles:

    cd ~/08-ci-cd/01-manual-release

List the current files:

    ls

You should expect at least the following files to exist:

- `index.html`
- `Dockerfile`
- `manual-web.yaml`

## Part 1: Understanding What Image Building Depends On

### Step 1: View the Current Application Content

Execute:

    cat index.html

This file represents the minimal application content that will be released.

### Step 2: View the Current Dockerfile

Execute:

    cat Dockerfile

The expected content is similar to:

    FROM nginx:1.27
    COPY index.html /usr/share/nginx/html/index.html
    EXPOSE 80

### Understanding to Be Established in This Part

In this experiment, image building depends on at least two things:

1. Application content: `index.html`
2. Building instructions: `Dockerfile`

In other words:

**The input for the image building phase = Application content + Dockerfile**

## Part 2: Modify the Content and Rebuild a New Image

### Step 1: Change the Page to Version v3

Execute:

    cat > index.html <<'EOF'
    <html>
      <head>
        <meta charset="utf-8">
        <title>manual release v3</title>
      </head>
      <body>
        <h1>Hello K8s CI/CD</h1>
        <p>version: v3</p>
      </body>
    </html>
    EOF

Check the content:

    cat index.html

You should see:

    version: v3

### What to Observe Now

This step only modifies the local file.

At this point, there are no changes yet in:

- The local image
- The Harbor image
- The Pod in K8s

This indicates that:

**Changing the content does not necessarily mean the image has been updated.**

## Part 3: Execute Image Building

### Step 1: Build a Local Image

Execute:

    docker build -t manual-web:v3 .

View the image:

    docker images | grep manual-web

### Expected Outcome

You should see something like:

    manual-web    v3

### Understanding to Be Established Here

A new local image has been created:

- Image name: `manual-web`
- Tag: `v3`

This means that:

- The latest content of `index.html` has been included in the image.
- The new image version has been successfully built on this machine.

## Part 4: Verify the Content by Running the Image Locally

### Step 1: Start the Container

Execute:

    docker run -d --name manual-web-v### Observing on the Harbor Page

At this point, you should see that in addition to the existing `v1`, `v2`, and `v3`, there is another tag under the same repository:

- `dev-a1b2c3d-101`

### Current Understanding Required

This section introduces a more production-oriented approach to image management:

- It's not just about simple labels like `v1/v2/v3`.
- Instead, you can directly trace back to the source of construction using tags.

## Part 11: Comparing Several Common Tag Styles

You don't need to adopt all of these styles, but it's important to understand when each one is suitable.

### Method 1: `latest`

Example:

    latest

Suitable for:

- Temporary testing during the initial learning phase
- Quick local verification

Not suitable for:

- Long-term official releases
- Production rollbacks
- Precise troubleshooting

### Method 2: Manual Version Numbers

Example:

    v1
    v2
    v3

Suitable for:

- Current learning stages
- Demonstrating version changes manually

Issues:

- It's impossible to determine the source code or build origin.

### Method 3: Only Branch Names

Example:

    dev
    main

Suitable for:

- Indicating which development branch the image belongs to

Issues:

- The same branch might be overwritten repeatedly, making it less precise.

### Method 4: Branch Name + Commit ID

Example:

    dev-a1b2c3d

Suitable for:

- Tracking specific code commits

Limitation:

- It's difficult to distinguish between multiple pipeline reruns for the same commit.

### Method 5: Branch Name + Commit ID + Pipeline ID

Example:

    dev-a1b2c3d-101

Suitable for:

- More production-oriented
- Allows you to locate both the code source and specific pipeline execution records

It is highly recommended to establish this understanding at this stage.

## Part 12: Recommended Tag Strategies for This Phase

Based on your current learning level, it's suggested to use tags in two layers:

### Learning Layer

Continue using:

- `v1`
- `v2`
- `v3`

These are the most intuitive and help you observe version changes easily.

### Production Understanding Layer

Add another tag that is more production-oriented, such as:

    dev-a1b2c3d-101

This way, you won't be overwhelmed by complex naming at the beginning and can gradually develop a production-oriented perspective.

In other words, the most suitable approach for this phase is to:

**Maintain both “learning-type tags” and “production-type tags” for the same image.**

## Part 13: Revisiting the Release Chain from the Perspective of Tags

Now, let's place tags back into the entire release chain. You should understand it as follows:

### Application Content Phase

You make changes to:

- `index.html`

### Image Building Phase

You execute:

- `docker build -t manual-web:v3 .`

### Image Tagging Phase

You then execute:

- `docker tag manual-web:v3 manual-web:dev-a1b2c3d-101`

### Image Storing Phase

You execute:

- `docker push harbor.../manual-web:dev-a1b2c3d-101`

In other words, tags are not an independent concept but a crucial step after the image is built and before it is stored in the repository.

This is also why they are discussed in detail in Chapter 3.

## Part 14: This Section's Exercises

### Exercise 1: Simulate a Set of New Production-Style Tags Yourself

Assume:

- Branch name: `main`
- Commit ID: `f6e7d8c`
- Pipeline ID: `205`

Execute:

    docker tag manual-web:v3 manual-web:main-f6e7d8c-205

Then check:

    docker images | grep manual-web

### Exercise 2: Push These Tags to Harbor

Execute:

    docker tag manual-web:v3 your-Harbor-domain/test/manual-web:main-f6e7d8c-205
    docker push your-Harbor-domain/test/manual-web:main-f6e7d8c-205

Confirm whether the tags appear on the Harbor page.

### Exercise 3: Answer the Following 3 Questions

1. Why is `latest` not suitable as a long-term unique version identifier?
2. What problem does the commit ID solve?
3. Why is the pipeline ID valuable in production?

If you can explain these three questions independently, you have mastered the core content of this section.

## Key Points to Understand from This Section

After completing the exercises, it is recommended that you be able to clearly articulate the following:

- The input for