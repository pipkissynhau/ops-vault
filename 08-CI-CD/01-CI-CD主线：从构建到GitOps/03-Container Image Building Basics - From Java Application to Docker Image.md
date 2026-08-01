# 03-Container Image Building Basics: From Java Programs to Docker Images

## Documentation Notes

This article is the third note in the 08-CI-CD learning path.

This article continues using the learning approach from the previous two articles, based on the existing experimental environment. Through practical operations, it aims to understand what the "image building phase" actually does, and introduces more production-relevant image Tag design methods at appropriate positions.

The default environment for this article is as follows:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Harbor private registry
- `test` namespace as the experimental environment

This article does not directly introduce Jenkins, GitLab CI, or Helm for now. Instead, it focuses on clarifying the following main line first:

Application content  
→ Dockerfile  
→ Local image  
→ Image Tag  
→ Tracable image version in Harbor

## Tags

#Kubernetes #CI-CD #Docker #Harbor #MirrorBuild #Dockerfile #MirrorTag #commitid #pipelineid #I'llTakeYourNotes.

## Learning Objectives

After completing this article, you should master:

1. Understanding what inputs and outputs the image building phase has
2. Understanding the role of Dockerfile in image building
3. Understanding why local file modifications require re-building to enter a new image
4. Understanding why image tags shouldn't long-term use `latest`
5. Understanding what problems branch names, commit IDs, and pipeline IDs solve respectively
6. Being able to manually simulate a more production-relevant image tag method
7. Being able to explain "how a particular image version was built"

## Main Line of This Article

This article is divided into two parts:

### Part 1: How Images Are Built

Continue the experiment with a minimal static page image, focusing on observing:

- Modify content
- Build image
- Local runtime verification

### Part 2: Designing Tags After Image Building

Use a more production-relevant approach to apply different tag styles to the same image, and understand:

- `latest`
- `v3`
- `dev`
- `dev-commit`
- `dev-commit-pipeline`

and what scenarios they are suitable for.

## Experimental Preparation

Continue using the experiment directories from the previous two articles:

    cd ~/08-ci-cd/01-manual-release

Check current files:

    ls

Expected to at least contain:

- `index.html`
- `Dockerfile`
- `manual-web.yaml`

## Part 1: First Understand What Image Building Depends On

### Step 1: Check Current Application Content

Execute:

    cat index.html

This file represents the current minimal application content to be released.

### Step 2: Check Current Dockerfile

Execute:

    cat Dockerfile

Expected content similar to:

    FROM nginx:1.27
    COPY index.html /usr/share/nginx/html/index.html
    EXPOSE 80

### Understanding to Establish in This Section

In the current experiment, image building at least depends on two things:

1. Application content: `index.html`
2. Build instructions: `Dockerfile`

In other words:

**Image building phase input = Application content + Dockerfile**

## Part 2: Modify Content and Rebuild a New Image

### Step 1: Change the Page to v3

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

Check content:

    cat index.html

Expected to see:

    version: v3

### What to Observe at This Step

This step only modified the local file.

Nothing has happened yet:

- Local image changes
- Harbor image changes
- K8s Pod changes

This indicates:

**Content changes themselves do not equal image changes.**

## Part 3: Execute Image Building

### Step 1: Build Local Image

Execute:

    docker build -t manual-web:v3 .

Check image:

    docker images | grep manual-web

### Expected Phenomenon

Should see something like:

    manual-web    v3

### Understanding to Establish Here

Here we get a new local image:

- Image name: `manual-web`
- Tag: `v3`

This indicates:

- The latest content of `index.html` has been packaged into the image
- The new image version has been built and completed on this machine

## Part 4: Verify Image Content by Local Runtime

### Step 1: Start Container

Execute:

    docker run -d --name manual-web-v3 -p 18080:80 manual-web:v3

### Step 2: Access the Page

Execute:

    curl http://127.0.0.1:18080

### Expected Phenomenon

The output should include:

    version: v3

### Step 3: Delete Test Container

Execute:

    docker rm -f manual-web-v3

### Understanding to Establish Here

At this point, it indicates:

- New content has entered the image
- This image can run normally locally

Therefore, the image building phase is not "done after the build command", but should at least include:

1. Build the image
2. Local verification of the image content

## Part 5: Why "latest" Is Not Recommended for Production Use

First look at the simplest tag:

    manual-web:latest

Technically usable, but it will quickly cause problems in real release pipelines.

### Problem 1: Unable to Identify Image Source

You cannot directly know:

- Which branch it comes from
- Which commit it corresponds to
- Which build it belongs to

### Problem 2: Difficult to Rollback

If there's an issue on the line, only seeing `latest` makes it hard to immediately know which image was the previous version.

### Problem 3: Troubleshooting Difficulties

When a Pod uses:

    your-harbor/test/manual-web:latest

You only know it's the latest, but don't know:

- Who pushed this latest
- Which build it was from today
- Whether it was built from an error branch

Therefore:

**`latest` can be used as a temporary tag in the learning phase, but is not suitable as a long-term unique version identifier.**

## Part 6: How to Understand Image Tags at This Stage

First remember this sentence:

**The core role of a tag is to give an image an identifiable, traceable, and rollable version identity.**

In the learning phase, you've already used:

- `v1`
- `v2`
- `v3`

The advantage of this approach is:

- Simple
- Intuitive
- Easy to get started with

But it also has obvious limitations:

- Can't see the source branch
- Can't see the corresponding commit
- Can't see which pipeline build it belongs to

So starting from this article, we'll gradually shift our perspective from "manual version numbers" to "production traceable tags".

## Part 7: Three Common Types of Tag Information in Production

More production-like image tags typically contain three types of information:

### 1) Branch Name

Examples:

- `dev`
- `test`
- `main`
- `release`

It solves the problem of:

**Identifying which development branch this image comes from.**

Examples:

- `dev` indicates a development branch build
- `main` indicates a main branch build
- `release` indicates a release branch build

### 2) Commit ID

Examples of short commit hashes:

- `a1b2c3d`
- `9f8e7d6`

It solves the problem of:

**Precisely identifying which code commit this image corresponds to.**

This is a crucial tracking capability in production.

### 3) Pipeline ID

Examples:

- `101`
- `568`
- `2034`

It solves the problem of:

**Identifying which specific pipeline execution this image comes from.**

Why is this important?

Because the same commit may trigger multiple builds:

- First build fails
- Second build after environment fix
- Third build as a retry

Relying solely on commit ID may not accurately distinguish which pipeline execution produced the image.

## Part 8: Combining These Three Types of Information

There are many common combination patterns in production. For now, let's use the easiest to understand format:

    branch-name-commitid-pipelineid

Example:

    dev-a1b2c3d-101

This tag immediately shows:

- Branch: `dev`
- Commit: `a1b2c3d`
- Pipeline: `101`

### Advantages of This Format

1. Strong readability
2. Strong traceability
3. Easy to locate during rollback
4. Easy to correlate with CI platform logs

## Part 9: Manually Simulating Production-Style Tags in Current Experiment

Since this experiment isn't triggered by real GitLab CI/Jenkins, we'll manually simulate it first.

Assume:

- Branch name: `dev`
- Commit ID: `a1b2c3d`
- Pipeline ID: `101`

First, give the current local image `manual-web:v3` another production-style tag.

Execute:

    docker tag manual-web:v3 manual-web:dev-a1b2c3d-101

Check the image:

    docker images | grep manual-web

### Expected Outcome

You'll see the same image has multiple tags, for example:

- `manual-web:v3`
- `manual-web:dev-a1b2c3d-101`

### Current Understanding to Establish

These two tags point to the same image content, just with different naming conventions.

In other words:

- `v3` is more like a manual version number in the learning phase
- `dev-a1b2c3d-101` is more like a traceable identity in production

## Part 10: Continuing to Tag the Image with Harbor Production-Style Labels

Assume the Harbor repository address is:

    your Harbor domain/test/manual-web

Execute:

    docker tag manual-web:v3 your Harbor domain/test/manual-web:dev-a1b2c3d-101

Example:

    docker tag manual-web:v3 harbor.example.com/test/manual-web:dev-a1b2c3d-101

Then push:

    docker push your Harbor domain/test/manual-web:dev-a1b2c3d-101

### Observing on Harbor Page

At this point, you should see the same repository with, in addition to the original `v1`, `v2`, and `v3`, a new one:

- `dev-a1b2c3d-101`

### Current Understanding to Establish

We're now entering a more production-like image management approach:

- Not just simple `v1/v2/v3`
- But able to trace back the build source through the tag

## Part 11: Comparing Several Common Tag Styles

This section doesn't require you to adopt all of them, but understand their respective suitable scenarios.

### Style 1: latest

Example:

    latest

Suitable for:

- Early learning temporary testing
- Local quick validation

Not suitable for:

- Long-term official release
- Production rollback
- Precise troubleshooting

### Style 2: Manual Version Number

Example:

    v1
    v2
    v3

Suitable for:

- Current learning phase
- Manual demonstration of version changes

Issues:

- Can't see code source
- Can't see build source

### Style 3: Only Branch Name

Example:

    dev
    main

Suitable for:

- Indicating the development branch the image belongs to

Issues:

- Same branch will continuously overwrite, still not precise

### Style 4: Branch Name + Commit ID

Example:

    dev-a1b2c3d

Suitable for:

- Can trace to specific code commit

Limitation:

- Can't distinguish multiple pipeline reruns for the same commit

### Style 5: Branch Name + Commit ID + Pipeline ID

Example:

    dev-a1b2c3d-101

Suitable for:

- Closer to production
- Can locate both code source and specific pipeline execution record

This is the recommended understanding to establish at this stage.

## Part 12: Recommended Tag Strategy at This Stage

Based on your current learning status, we recommend using two layers.

### Learning Layer

Continue retaining:

- `v1`
- `v2`
- `v3`

Because it's the most intuitive, helping you observe version changes.

### Production Understanding Layer

Simultaneously apply a more production-like tag, for example:

    dev-a1b2c3d-101

This way, you won't be overwhelmed by the initially complex naming, while gradually building a production perspective.

In other words, the most suitable approach at this stage is:

**A single image retains both "learning-type tag" and "production-type tag" awareness.**

## Part 13: Revisiting the Release Chain from the Perspective of Tags

Now place the tag back into the full release chain. You should understand it this way:

### Application Content Stage

You modified:

- `index.html`

### Image Build Stage

You executed:

- `docker build -t manual-web:v3 .`

### Image Identification Stage

You also executed:

- `docker tag manual-web:v3 manual-web:dev-a1b2c3d-101`

### Image Storage Stage

You executed:

- `docker push harbor.../manual-web:dev-a1b2c3d-101`

In other words, tags aren't independent additional knowledge points, but:

**After the image is built, but before pushing it to the registry, this is a critical step.**

This is why it's most appropriate to place it in Part 3.

## Part 14: This Chapter's Exercise

### Exercise 1: Simulate a New Production-Style Tag Yourself

Assume:

- Branch name: `main`
- Commit ID: `f6e7d8c`
- Pipeline ID: `205`

Execute:

    docker tag manual-web:v3 manual-web:main-f6e7d8c-205

Then check:

    docker images | grep manual-web

### Exercise 2: Push This Tag Group to Harbor

Execute:

    docker tag manual-web:v3 your-Harbor-domain/test/manual-web:main-f6e7d8c-205
    docker push your-Harbor-domain/test/manual-web:main-f6e7d8c-205

Confirm the tag appears on the Harbor page.

### Exercise 3: Answer the Following 3 Questions

1. Why is `latest` not suitable as a long-term unique version identifier?
2. What problem does the commit ID solve?
3. Why is the pipeline ID valuable in production?

If you can explain these 3 questions yourself, you've mastered the core of this chapter.

## Content You Should Be Able to Explain After This Chapter

After completing the experiment, it's recommended to be able to explain the following:

The input to the image build phase is application content and Dockerfile, and the output is a local image.  
When local files are modified, you must re-execute the build for new content to enter the new image.  
After the image is built, it's also necessary to assign a reasonable tag to the image.  
During the learning phase, you can first use `v1/v2/v3` this type of simple tag to observe version changes, but in production it's recommended to use tags containing branch name, commit ID, and pipeline ID, such as `dev-a1b2c3d-101`.  
The benefit of this approach is that images become traceable, locatable, androllable, and it's easier to align with subsequent GitLab CI or Jenkins pipelines.

## Common Issues and Troubleshooting

### Issue 1: Changed index.html, but the container content hasn't changed

The usual reasons are:

- Not rebuilding
- Still running the old tag container

Check:

    docker images | grep manual-web
    docker run -d --name test1 -p 18080:80 manual-web:v3
    curl http://127.0.0.1:18080

### Issue 2: Why can the same image have multiple tags?

Because tags are essentially "image reference names," not completely independent new image content.

### Issue 3: Why is the pipeline ID still needed for the same commit?

Because the same commit may trigger multiple builds, the pipeline ID helps further identify which specific build task it is.

## Key Takeaways from This Chapter

After completing this chapter, you should understand:

1. Inputs and outputs of the image build phase
2. The role of Dockerfile in the build process
3. Why rebuilding is mandatory after application content changes
4. Why long-term use of `latest` is not recommended in production
5. What branch name, commit ID, and pipeline ID represent respectively
6. How to manually simulate more production-like image tags
7. Why tag design belongs to an important extension of the image build phase

## One-Sentence Summary

The image build phase isn't just about packaging application content into an image - it also requires reasonable tag design to give images production attributes of being traceable, locatable, androllable.

## Next Chapter

Next chapter will enter:

04-Harbor Basics: Projects, Repositories, Tags, and Robot Accounts

The next chapter will continue from this chapter's tag focus, shifting to the Harbor perspective to understand:

- What projects, repositories, and tags are in Harbor
- How to view and manage multiple tags in the same repository
- Why Robot accounts are suitable for GitLab CI/Jenkins usage
- How this will connect with K8s pulling private images from Harbor in the future