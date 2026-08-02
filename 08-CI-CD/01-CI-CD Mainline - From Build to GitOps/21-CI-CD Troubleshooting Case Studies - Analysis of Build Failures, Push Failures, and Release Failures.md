# 21-CI-CD Troubleshooting Case Studies: Analysis of Build Failures, Push Failures, and Release Failures

## Document Description

This article is the 21st note in the 08-CI-CD learning pathway.

In the previous 01-20 notes, we successfully established a minimal release pipeline and gradually covered the following aspects:

- Manual release
- Image building and tagging
- Harbor repository
- GitLab CI/Jenkins Pipeline
- Deployment rolling updates
- Helm
- Multi-environment releases
- Executors
- Security fundamentals
- Advanced Harbor usage
- GitOps basics
- Blue-green, canary, and grayscale deployment strategies

In this article, the focus shifts from "how to release" to:

**How to troubleshoot release failures step by step along the pipeline.**

Instead of general discussions about "checking logs and analyzing causes," we will specifically address common faults within the minimal pipeline you have already set up, categorizing them into three main types:

1. Build failures
2. Push failures
3. Release failures

We will also try to provide examples that can be directly replicated and observed in your current environment.

This article continues to use the following setup:

- Ubuntu 22.04
- Kubernetes v1.31.x
- containerd 2.x
- Private Harbor repository
- `test` namespace
- Existing minimal application: `manual-web`

## Tags

#Kubernetes #CI-CD #Troubleshooting #Docker #Harbor #Deployment #Helm #rollout #ImagePullBackOff #Practical Notes

## Learning Objectives

After completing this article, you should be able to:

1. Understand why it is essential to troubleshoot issues step by step along the pipeline.
2. Know where to look for specific problems in build failures, push failures, and release failures.
3. Replicate several common faults in your current environment.
4. Quickly identify which stage of the pipeline the issue is likely occurring at based on the symptoms.
5. Clearly explain the troubleshooting sequence for a minimal release pipeline.

## Main Troubleshooting Approach

For now, let's divide potential issues into three major segments:

### Segment 1: Build Failures
These occur during:

- Local build processes
- Runner/Agent build processes
- The Dockerfile processing stage

### Segment 2: Push Failures
These issues arise when:

- Attempting to log in to Harbor
- Tagging errors occur
- Repository permissions are incorrect
- There are connection problems with the repository

### Segment 3: Release Failures
These happen during:

- Deployment/Helm update processes
- Pod image pulling
- Container startup
- Rollout convergence
- Business validation steps

Once we identify these three segments, many subsequent issues will become much clearer.

---

## Part 1: Establish a Standard Approach to Step-by-Step Troubleshooting

The current minimal release pipeline consists of the following steps:

Application content → Image building → Image storage in Harbor → Cluster configuration → Cluster deployment → Release validation

When troubleshooting, avoid making vague statements like:

- "CI/CD failed"
- "There's a problem with K8s"
- "Harbor can't retrieve the image"

A better approach is to first determine:

### 1) Whether the build phase was successful
Examples include:

- Errors in the Dockerfile
- Incorrect build context settings
- The local image cannot be generated

### 2) Whether the image was successfully uploaded to Harbor
Possible issues include:

- Push failures
- Tagging errors
- Insufficient Harbor permissions

### 3) Whether the cluster is properly using the image
Potential problems include:

- Incorrect image names in the Deployment configuration
- The Pod cannot retrieve the image
- Rollout processes are stuck
- The web page still displays old content

### Key Understanding for This Step

The first step of troubleshooting is to break down the issue into smaller, more manageable segments.

---

## Part 2: Build Failures – Focus on the Build Phase First, Don’t Assume Issues with Harbor or K8s Yet

Problems in this phase usually occur during:

- `docker build` commands
- Local image verification steps
- The Dockerfile itself
- Build context settings

### Common Symptoms

- Immediate errors during `docker build`
- Successful build but incorrect content when running locally
- Successful build but failed container startup

---

## Part 3: Troubleshooting Case Study 1 – Incorrect File References in the Dockerfile

### Issue Objective

Simulate a common build failure scenario:

**The Dockerfile contains a COPY command referencing a non-existent file.**

### Step 1: Enter the experiment directory

    cd ~/08-ci-cd/01-manual-release

### Step 2: Intentionally modify the Dockerfile

Replace the original line:

    COPY index.html /usr/share/nginx/html/index.html

with an incorrect version:

- The rollout is stuck.
- The page still shows the old content.The issue of "the page still being the old version" cannot be solely blamed on K8s, as problems with content, build processes, pushes, image references, Helm values, or Service paths can all ultimately result in the same outcome.

## Common Issues and Troubleshooting Approaches

### Issue 1: Why do similar "release failures" have such different causes?
This is because release is just the latter half of the process; errors in the earlier stages of build or push can also cause subsequent issues.

### Issue 2: Despite knowing many commands, why am I still easily confused when troubleshooting?
Commands are just tools; true stability comes from "identifying the problem at each stage."

### Issue 3: Are Helm-related issues harder to troubleshoot?
Not necessarily, but there are additional layers to consider compared to using kubectl directly:
- Values
- Template rendering
- Release history

Therefore, it's important to use tools like:
- `cat values`
- `helm template`
- `helm history`

---

## Key Points to Remember from This Article

1. The basic stages of build failure, push failure, and release failure.
2. Typical symptoms of common faults.
3. Priority troubleshooting directions for different issues.
4. The most effective order of troubleshooting at this stage.
5. How to improve from being able to perform releases to being able to accurately locate problems.

## One-Sentence Summary

The key to CI/CD troubleshooting is not simply checking logs after a failure occurs, but rather breaking down the problem into stages such as "build", "storage", "release", and "verification" and then locating the issue step by step along the process.